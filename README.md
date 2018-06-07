# Local SonarQube with .NET Core
This repository contains instructions for:
- Running the SonarQube server locally in Docker 
- Analyzing a .NET Core 2.* project with SonarQube and OpenCover

## About SonarQube
[SonarQube - Continuous Code Quality.](https://www.sonarqube.org/) The SonarQube continuous quality platform allows engineering teams to have a continuous *objective* quality measure of systems as they are being developed. This allows teams to monitor the overall health of code-bases as new features are added. The platform is meant to run as a step in a system's continuous integration pipeline.

Quality is assessed through static analysis rules. SonarQube scans a codebase during the build process and captures the results of static analysis rules. These rules can be enforced on a per-project basis. The rules are also customization on a per-project basis. Static analysis results are stored on a server and are accessible through a web browser.

SonarQube can also aggregate code-coverage results from coverage tools as a part of a continuous quality cycle.

SonarQube has comprehensive documentation - [SonarQube docs](https://docs.sonarqube.org/display/SONAR/Documentation)

## SonarQube Structure
Continuous quality with SonarQube can be broken into three separate pieces: SonarQube Server, SonarQube Scanners, and CI updates.

SonarQube is an actively developed open-source project. Updates can be found on the [SonarSource blog](https://blog.sonarsource.com/).

### SonarQube Server
The main, visible, piece of SonarQube is a server to that aggregates quality results in a centralized location. The server provides aggregate information across all codebases, as well as detailed analysis of issues and bugs discovered through code analysis. Code smells, bugs, and security vulnerabilities can be detected through SonarQube. SonarQube uses Quality Gates to ensure that code added to a project does not degrade the quality of the entire system.

#### Running the SonarQube Server Locally
Assuming that docker has already been installed on the target machine.

1. Pull the latest version of this repository.
1. In a administrator instance of PowerShell, type `docker-compose up` to run a local instance of the SonarQube Server.
1. Navigate to SonarQube, running on `localhost:9000` 
Running a local version of the SonarQube allows developers to easily test configuration of CI steps.

The default authentication credentials are username: `admin`, password: `admin`.

### SonarQube Scanners
Scanners wrap existing software builds and collect quality data about the system that is being built. SonarQube supports many different Scanners for different build platforms, but the SonarQube Scanner for MSBuild will be the most commonly used Scanner.

#### Build Agent Configuration for .NET Projects
[This page](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+MSBuild) contains installation and configuration instructions for the MSBuild SonarQube Scanner. The page also details usage instructions for the scanner. 

The MSBuild Scanner can also be used with .NET Core 2.* projects. Wrapping the build command in the scanner's `begin` and `end` commands sends the output of the quality analysis to the SonarQube server that is is defined in the scanner's configuration. 

**Additional Configuration**
Verify that the `MSBuild.SonarQube.Runner.exe` command is available from an admin instance of PowerShell. 

Configure the Path system environment variable to contain a reference to MSBuild and the `MSBuild` command. Use the path to `MSBuild.exe` from a Visual Studio installation 
(ex. `C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\MSBuild\15.0\Bin`).

Configure the Path system environment variable to contain a reference to Java and the `Java` command. 


#### Build Agent Configuration for OpenCover (.NET code code-coverage)
[OpenCover](https://github.com/OpenCover/opencover) is a .NET code coverage tool thats output can be read by the SonarQube platform. OpenCover needs to be installed on Build Agents in order to perform code-coverage analysis and include the results in the SonarQube platform.

1. Download the latest release of OpenCover (.zip file) - [Download Page](https://github.com/opencover/opencover/releases)
1. Extract the contents to a known location on the Build Agent.
1. Edit the PATH environment variable with a reference to the OpenCover executable.

### Updates to CI Configurations
The following is a sample PowerShell script that can be run in a CI pipeline. This script contains commands to wrap the build of a project in SonarQube scanners. It also contains steps to run code coverage on Unit test projects with OpenCover.

```
param (
  [Parameter(Mandatory=$true)][string]$key,
  [Parameter(Mandatory=$true)][string]$name,
  [Parameter(Mandatory=$true)][string]$version,
  [string]$coverage_directory = ".\coverage\coverage.xml"
)

# SonarQube begin command, key name and version read from the command line
SonarQube.Scanner.MSBuild.exe begin /k:$key /n:$name /v:$version /d:sonar.cs.opencover.reportsPaths=$coverage_directory

# The build itself (very basic...)
MSbuild.exe /t:Rebuild

# OpenCover commands, one for every unit test project
OpenCover.Console.exe -target:"C:\Program Files\dotnet\dotnet.exe" -targetargs:"test -f netcoreapp2.0 -c Debug path\to\project\.csproj" -mergeoutput -hideskipped:Filter -oldstyle -output:coverage\coverage.xml -filter:"+[*]* -[*.UnitTests*]*" -searchdirs:"path\to\project\debug\directory\" -register:user
# Repeat above for each project to analyze

#SonarQube end command
SonarQube.Scanner.MSBuild.exe end
```
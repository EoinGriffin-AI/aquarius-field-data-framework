﻿# Field Visit Hot Folder Service

Download the latest version of the Field Visit Hot Folder Service [from the releases page](https://github.com/AquaticInformatics/aquarius-field-data-framework/releases/latest).

The `FieldVisitHotFolderService.exe` tool is a .NET console tool which can:
- Run from the command line and Ctrl-C to exit,  or run as a Windows service.
- Monitor a hot folder path for new files
- Try to parse the files using a set of plugins running locally on the client
- Upload any created visits to AQTS using the [JSON plugin](../JsonFieldData/Readme.md)

The service can be run on any Windows system with the .NET 4.7.2 runtime. This includes every Windows 10 desktop and Windows 2016 Server, and any Windows 7 desktop and Windows 2008 R2 Server with the most recent Windows Update patches.

# Installing the service

- Download the `FieldVisitHotFolderService.zip` archive from the [releases page](https://github.com/AquaticInformatics/aquarius-field-data-framework/releases/latest) and unzip the contents into a folder.
- Run the `InstallService.cmd` batch file with elevated privileges to install the service.
- Run the `UninstallService.cmd` batch file to uninstall the service.
- The service expects an `Options.txt` file to exist in the same folder as the EXE.
- The `Options.txt` file uses the [@options.txt syntax](https://github.com/AquaticInformatics/examples/wiki/Common-command-line-options) to store its configuration options.

# Configuring the service

Create an `Options.txt` file in the same folder as the EXE, to contain all the configuration options.

The following 12 lines are a good start for the `Options.txt` file:
```
# Enter your AQTS credentials here.
# The account should have "CanAddData" permission.
/Server=myAppServer
/Username-hotfolderuser
/Password=pass123

# Configure the hot folder path to monitor
/HotFolderPath=D:\Some Folder\Drop Field Visit Files Here

# Configure the local plugins to permit (you need at least one)
/Plugin=C:\MyPlugins\FlowTracker2
/Plugin=C:\MyPlugins\StageDischargeReadings
```

You can test your configuration without installing the service just by running the EXE directly from the command line or by double-cliking it from Windows Explorer.

When not run as Windows Service, the program will run until you type Ctrl-C or Ctrl-Break to exit.

## Folder configuration

There are five configurable folders which are used to process field visit files.

- The `/HotFolderPath` option is the folder that will be watched for new files.
- When a new file is detected, it is moved to the `/ProcessingFolder` while it is being processed.
- After processing, the file will be moved to the `/UploadedFolder` if it can successful upload all of its results to AQTS.
- The file will be moved to the `/PartialFolder` which at least one visit was skipped with a `WARN` to avoid duplicates.
- Otherwise processed file will be moved to the `/FailedFolder` if it fails to upload anything to AQTS.

The `/ProcessingFolder`, `/UploadedFolder`, `/PartialFolder`, and `/FailedFolder` can be absolute paths, or can be paths relative to the base `/HotFolderPath` folder. The folders will be created if they don't already exist.

- These folders can be local to the computer running `FieldVisitHotFolderService.exe`, or can be a UNC network path.
- The program will need file system rights to read, write, delete, and move files in all of these folder locations.

# Operation

- The service will monitor the `HotFolderPath` for files matching the `FileMask` pattern.
- When a new file is detected, the service will wait for the `FileQuietDelay`, before attempting to process the file.
- The file content will be loaded into memory, and passed to each `/Plugin` option, until a plugin returns a successful parse result.
- For every successfully parsed visit, the service will check if a visit already exists on that day at the requested location. These "conflicting visits" will be skipped with a `WARN` log line, but will not cause a failure.
- If no plugins can parse the file successfully, or if an upload error occurs, the file will be considered to have failed, and will be moved to the `/FailedFolder`.

## Conditions which can cause a file to be "failed"

Any of these conditions will cause the file to be considered failed, and will be moved to the `/FailedFolder`:
- The file was not recognized by one of the local `/Plugin` parsers.
- The location identifiers referenced in the file do not exist on the AQTS system.
- A validation error occurred when the visit was uploaded to AQTS.

## "Partial uploads" - Existing visits on the same day cannot be overwritten

AQTS does not allow 2 visits on the same day in a location, and does not allow visit data from a plugin to be merged with a visit already existing in AQTS.

So before the service uploads a locally-parsed visit, it checks if the location already has a visit on the same day.
If an existing visit is detected, the service will not attempt to upload the conflicting visit.
The upload will be skipped and logged with a `WARN` line, but will not cause the file processing to fail.
The file will be moved to the `/PartialFolder`.

This feature allows the hot folder to consume files which are repeatedly generated via an append operation, containing only new data appended to the end of the file.

## Detecting a valid AQTS app server connection

When the service is started, and immediately before processing any detected files, the following logic will be applied:

- If no local plugins are configured with the `/Plugin=` option, the service will exit.
- If the AQTS app server is not running, the service will wait for the `ConnectionRetryDelay`, for up to `MaximumConnectionAttempts`.
- If the maximum connection attempts is reached without a successful connection, the service will stop.
- If `MaximumConnectionAttempts` is less than 1, the service will wait repeatedly until the app server is running, or the service is manually stopped.
- Once a valid AQTS connection is established, a few more configuration inspections are made.
- If the AQTS server is not running AQTS 2018.4-or-newer, the service will exit.
- If the AQTS server does not have the [JSON field data plugin](../JsonFieldData/Readme.md) installed, the service will exit.

# Log files

The service creates a `FieldVisitHotFolderService.log` file in the same directory as the EXE.

In addition, each processed file will have a `{filename}.log` file in the appropriate `/UploadedFolder`, `/PartialFolder`, or `/FailedFolder`, for debugging purposes.

# /Help screen

```
Purpose: Monitors a folder for field visit files and appends them to an AQTS app server

Usage: FieldVisitHotFolderService [-option=value] [@optionsFile] ...

Supported -option=value settings (/option=value works too):

  =========================== AQTS connection settings
  -Server                     The AQTS app server.
  -Username                   AQTS username. [default: admin]
  -Password                   AQTS credentials. [default: admin]
  -MaximumConnectionAttempts  The maximum number of connection attempts before exiting. [default: 3]
  -ConnectionRetryDelay       The TimeSpan to wait in between AQTS connection attempts. [default: 00:01:00]

  =========================== Local plugin settings
  -Plugin                     A plugin assembly to use for parsing field visits locally. Can be set multiple times.

  =========================== File monitoring settings
  -HotFolderPath              The root path to monitor for field visit files.
  -FileMask                   A comma-separated list of file patterns to monitor. [defaults to '*.*' if omitted]
  -FileQuietDelay             Timespan of no file activity before processing begins. [default: 00:00:05]
  -ProcessingFolder           Move files to this folder during processing. [default: Processing]
  -UploadedFolder             Move files to this folder after successful uploads. [default: Uploaded]
  -PartialFolder              Move files to this folder if when partial uploads are performed to avoid duplicates. [default: PartialUploads]
  -FailedFolder               Move files to this folder if an upload error occurs. [default: Failed]

Use the @optionsFile syntax to read more options from a file.

  Each line in the file is treated as a command line option.
  Blank lines and leading/trailing whitespace is ignored.
  Comment lines begin with a # or // marker.
```
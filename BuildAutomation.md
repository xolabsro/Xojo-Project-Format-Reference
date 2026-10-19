# Xojo Build Automation — Format Reference

Every Xojo project has a `Build Automation.xojo_code` file. **The IDE creates it when the project is created.**
This page explains what the file contains, so
you can read an existing one and understand what happens before and after a build.

The manifest lists it as
`BuildSteps=Build Automation;Build Automation.xojo_code;<ItemID>;&h0000000000000000;false`. It holds no code: it's
a `#tag BuildAutomation` … `#tag EndBuildAutomation` block with a `BuildStepList` for each build target.

**You can edit steps inside the existing lists.** You can add, change, reorder or remove steps such as Copy
Files. Leave the `BuildStepList` lines, `BuildProjectStep Build` and `SignProjectStep Sign` in place.
Match the format described here exactly.

## What the file looks like

```
#tag BuildAutomation
			Begin BuildStepList <Target>
				Begin <StepType> <StepName>
					<Property> = <Value>
				End
			End
#tag EndBuildAutomation
```

- **Indentation:** tabs only. `BuildStepList` lines have 3 tabs, the steps inside them 4, and step properties 5.
  `SignProjectStep` is the exception: its properties have 4 tabs plus two spaces, with no spaces around `=`.
  Its properties vary by project type: Desktop and Console have `DeveloperID` and `macOSEntitlements`, Web
  and Android only `DeveloperID`, and iOS none.
- **Targets** (the text after `BuildStepList`) depend on the project type. These are the target names that can
  appear in each one, in the order the IDE writes them:

  | Project   | Targets                                      |
  | --------- | -------------------------------------------- |
  | Desktop   | `Linux`, `Mac OS X`, `Windows`               |
  | Console   | `Linux`, `Mac OS X`, `Windows`               |
  | Web       | `Linux`, `Mac OS X`, `Windows`, `Xojo Cloud` |
  | Android   | `Linux`, `Mac OS X`, `Windows`, `Android`    |
  | iOS       | `Linux`, `Mac OS X`, `Windows`, `iOS`        |

  A file doesn't have to have every list. A file with only the `iOS` list loads fine (see the iOS example under
  Copy Files step).
- **Order is execution order.** Each list has one `BuildProjectStep Build`, which is the compile itself. Steps
  above it run before the build, and steps below it run after. On `Mac OS X` and `iOS` a `SignProjectStep Sign`
  follows `Build`. Copy steps come before `Sign` so the copied files get signed.
- **Step names** (the text after the step type) are unique within the project. They're the names shown in the
  IDE's navigator. The IDE names new steps `CopyFiles1`, `CopyFiles2`, … and the user may rename them, for
  example `CopyFilesBeforeBuildMacOS`.

## Copy Files step

A Copy Files step copies files or folders into the built app. The example below is IDE output from a Desktop project. It has Copy Files steps before and after the build on macOS and Windows; the user renamed the steps and added files to the "before" steps:

```
#tag BuildAutomation
			Begin BuildStepList Linux
				Begin BuildProjectStep Build
				End
			End
			Begin BuildStepList Mac OS X
				Begin CopyFilesBuildStep CopyFilesBeforeBuildMacOS
					AppliesTo = 0
					Architecture = 0
					Target = 0
					Destination = 0
					Subdirectory = 
					FolderItem = Li4vUkVBRE1FLm1k
					FolderItem = Li4vLi4vTElDRU5TRQ==
				End
				Begin BuildProjectStep Build
				End
				Begin CopyFilesBuildStep CopyFilesAfterBuildMacOS
					AppliesTo = 0
					Architecture = 0
					Target = 0
					Destination = 0
					Subdirectory = 
				End
				Begin SignProjectStep Sign
				  DeveloperID=
				  macOSEntitlements={"App Sandbox":"False","Hardened Runtime":"False","Notarize":"False","UserEntitlements":""}
				End
			End
			Begin BuildStepList Windows
				Begin CopyFilesBuildStep CopyFilesBeforeBuildWindowsOS
					AppliesTo = 0
					Architecture = 0
					Target = 0
					Destination = 0
					Subdirectory = 
					FolderItem = Li4vLi4vUkVBRE1FLm1k
				End
				Begin BuildProjectStep Build
				End
				Begin CopyFilesBuildStep CopyFilesAfterBuildWindowsOS
					AppliesTo = 0
					Architecture = 0
					Target = 0
					Destination = 0
					Subdirectory = 
				End
			End
#tag EndBuildAutomation
```

An iOS project whose file has only the `iOS` list, with a new Copy Files step that still has its default name and values:

```
#tag BuildAutomation
			Begin BuildStepList iOS
				Begin CopyFilesBuildStep CopyFiles1
					AppliesTo = 0
					Architecture = 0
					Target = 0
					Destination = 0
					Subdirectory = 
				End
				Begin BuildProjectStep Build
				End
				Begin SignProjectStep Sign
				End
			End
#tag EndBuildAutomation
```

The IDE always writes the properties in this order as `Name = Value`, even when a value is `0` or empty:

| Property       | Default | Meaning                                                                                    |
| -------------- | ------- | ------------------------------------------------------------------------------------------ |
| `AppliesTo`    | `0`     | When the step runs: `0` = Both, `1` = Debug, `2` = Release, `3` = None (disabled)          |
| `Architecture` | `0`     | `0` = Any, `1` = Intel, `2` = ARM. `0` is the usual value.                                 |
| `Target`       | `0`     | Only `0` has been seen.                                                                    |
| `Destination`  | `0`     | Where the files go: `0` to `4`, see the table below                                        |
| `Subdirectory` | empty   | A folder created under the destination to hold the files. Empty means `Subdirectory = ` (with a trailing space) and the files go straight into the destination. |
| `FolderItem`   | none    | A file or folder to copy, one line each. A step with no `FolderItem` lines copies nothing. |

`Destination` values on each platform:

| Value | Name                 | Windows / Linux                   | macOS                               |
| ----- | -------------------- | --------------------------------- | ----------------------------------- |
| `0`   | App Parent Folder    | next to the executable            | `Contents/MacOS`                    |
| `1`   | Resources Folder     | `Resources` folder beside the app | `Contents/Resources`                |
| `2`   | Frameworks Folder    | `Libs` folder                     | `Contents/Frameworks`               |
| `3`   | Bundle Parent Folder | folder containing the executable  | folder containing the `.app` bundle |
| `4`   | Contents Folder      | folder containing the executable  | the bundle's `Contents`             |

`FolderItem` values are **Base64-encoded relative paths**. To decode one, run
`printf '%s' 'Li4vLi4vTElDRU5TRQ==' | base64 -d`. To encode one, run `printf '%s' '../../LICENSE' | base64`. Use
`printf`, not `echo`, so no newline gets encoded. The values in the Desktop example above:

| Value                  | Decodes to        |
| ---------------------- | ----------------- |
| `Li4vUkVBRE1FLm1k`     | `../README.md`    |
| `Li4vLi4vUkVBRE1FLm1k` | `../../README.md` |
| `Li4vLi4vTElDRU5TRQ==` | `../../LICENSE`   |

The path is relative to the `.xojo_project` file itself, not to its folder: the first `../` leaves the file. For
a project at `MyApp/MyApp.xojo_project`, `../README.md` is `MyApp/README.md`, and `../../LICENSE` is the
`LICENSE` in the folder that contains `MyApp/`.

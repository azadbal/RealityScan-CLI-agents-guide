## Agent Overview

This top section is a distilled agent-facing guide synthesized from the official RealityScan / RealityCapture CLI docs. It is intended to give an agent the operating model first, before the merged raw help reference below.

# RealityScan / RealityCapture CLI & RCCMD Knowledge Map

This document is a **mental model + reference map** distilled from the official RealityScan / RealityCapture CLI PDFs. It is intended to onboard another agent (human or AI) so they can reliably help write, reason about, and debug CLI and `.rscmd/.rccmd` workflows.

The focus is **how the system actually behaves**, not just command lists.

---

## 1. High-level execution model

RealityScan / RealityCapture CLI is **imperative and sequential**.

* The executable is invoked once.
* Commands are executed **in the exact order provided**.
* Each command mutates internal application state (scene, components, selection, settings).
* Later commands implicitly depend on earlier state.

This means:

* There is **no DAG** or dependency resolution.
* There is **no rollback**.
* If a command fails, later commands may run on invalid state unless error handling is enabled.

CLI is effectively a macro recorder for the GUI.

---

## 2. Three ways to issue commands

### 2.1 Direct CLI invocation

```bash
RealityScan.exe -newScene -addFolder "images" -align -save "scene.rsproj"
```

* Suitable for short, one-shot runs
* Hard to maintain for pipelines

### 2.2 `.rscmd` / `.rccmd` script files (core mechanism)

A plain-text file containing one command per line:

```txt
# comments allowed
-newScene
-addFolder "images"
-align
-save "scene.rsproj"
```

Executed via:

```bash
RealityScan.exe -execRSCMD "script.rccmd"
```

Key properties:

* No flow control (no if/loops)
* No variables except positional args
* No concept of script directory

This is the **primary automation primitive**.

### 2.3 Delegation to a running instance

Commands can be sent to a named, already-running instance.

Used for:

* Long-running background jobs
* Multi-instance orchestration
* External schedulers

---

## 3. Working directory and paths (important)

* **Relative paths are resolved against the process working directory**, not the `.rccmd` file location.
* There is **no token** like `$SCRIPT_DIR` or `%~dp0` inside `.rccmd`.

Implications:

* Always launch RealityScan from a wrapper (`.bat`, shell script) that `cd`s into a known directory.
* Or pass absolute paths via arguments.

Example wrapper pattern:

```bat
cd /d "%~dp0"
RealityScan.exe -execRSCMD "pipeline.rccmd"
```

---

## 4. Arguments and parameterization

`.rscmd/.rccmd` supports **positional arguments**:

* `$(arg1)` ... `$(arg9)`

Passed at execution time:

```bash
RealityScan.exe -execRSCMD pipeline.rccmd images output
```

```txt
-addFolder "$(arg1)"
-save "$(arg2)\\scene.rsproj"
```

There are:

* No named arguments
* No defaults
* No type checking

Arguments are **string substitution only**.

---

## 5. Project & scene lifecycle

### Core scene commands

* `-newScene` - resets everything
* `-load <project>` - loads existing project
* `-save <project>` - saves current state

Rules:

* Many commands require a loaded or new scene
* Loading implicitly discards unsaved state

---

## 6. Input ingestion

### Image input

* `-add <file>` - single image
* `-addFolder <folder>` - recursive image add

Additional capabilities:

* Enable/disable images
* Masking
* Layer selection

All ingestion affects **current scene only**.

---

## 7. Settings system (critical distinction)

There are **two fundamentally different setting mechanisms**.

### 7.1 `-set` (runtime state)

```txt
-set "appQuitOnError=true"
```

* Affects current session only
* Can be changed anytime
* Lost when app exits

### 7.2 `-preset` (pre-processing / setup)

```txt
-preset "alignmentQuality=High"
```

* Must be applied **before** the relevant computation
* Mirrors GUI preset changes
* Affects how future operations behave

Key rule:

> If a setting affects how alignment / reconstruction is *computed*, it must be a `preset`, not `set`.

---

## 8. Alignment and components

### Alignment commands

* `-detectFeatures`
* `-align`

Alignment creates **components**.

### Component handling

* Multiple components may exist
* Many downstream commands operate on the **selected component**

Common selection commands:

* `-selectMaximalComponent`
* `-selectComponent <id>`

Failing to select a component is a **common source of silent failure**.

---

## 9. Registration & camera exports

Registration export is unified:

```txt
-exportRegistration <output> <params.xml>
```

Key idea:

* **Format is not chosen in CLI**
* Format is defined in the **Export Registration XML**

COLMAP, XMP, internal formats all use the same command.

Workflow:

1. Configure export once in GUI
2. Save settings XML
3. Reuse via CLI

---

## 10. Reconstruction region

Reconstruction is **region-bounded**.

Region commands:

* Auto-generate region
* Import/export region
* Scale / rotate / move

Best practice:

* Explicitly define region before reconstruction
* Do not rely on GUI leftovers

---

## 11. Reconstruction (meshing)

Three main compute commands:

* `-calculatePreviewModel`
* `-calculateNormalModel`
* `-calculateHighModel`

Properties:

* Blocking operations
* Heavy GPU/CPU usage
* Can be resumed with `-continueModelCalculation`

Reconstruction always applies to:

* Selected component
* Current reconstruction region

---

## 12. Model tools (post-processing)

Operate on the **currently active model**:

* Simplify
* Smooth
* Cut
* Unwrap
* Texture
* Export model

These are destructive unless saved as new models.

---

## 13. Error handling & observability

### Silent / headless operation

```txt
-silent "crash_reports"
```

Prevents dialogs and UI blocks.

### Progress reporting

* `-printProgress`
* `-writeProgress <file>`

### Failure control

* `-set "appQuitOnError=true"`

Without this, failures may not stop execution.

---

## 14. Delegation & multi-instance control

### Instance naming

```txt
-setInstanceName worker1
```

### Delegation

```txt
-delegateTo worker1 -align
```

### Synchronization

* `-waitCompleted`
* `-getStatus`
* `-abortInstance`

Used for:

* Job queues
* Farm-like orchestration
* External schedulers

---

## 15. Commands outside command prompt

Some operations:

* Rely on OS integration
* Are triggered by drag-drop or file association

CLI generally bypasses these.

---

## 16. Common failure modes

* Forgetting `-selectMaximalComponent`
* Using `-set` instead of `-preset`
* Relative paths resolving incorrectly
* Export commands relying on GUI state not recreated in CLI
* Assuming formats are CLI flags (they are not)

---

## 17. Mental model summary

Think of RealityScan CLI as:

> A deterministic, stateful, single-threaded macro engine that replays GUI actions exactly as if a user clicked them - nothing more, nothing less.

There is:

* No abstraction layer
* No schema validation
* No smart defaults

Correct automation comes from:

* Explicit state setup
* Explicit selection
* Saved XML parameter reuse
* External wrappers for structure

---

## 18. How to help write commands effectively

When helping with CLI:

Always ask:

1. What scene state exists at this point?
2. What is currently selected?
3. Which settings must be presets vs runtime?
4. Which GUI dialog defines this behavior?
5. Is there a saved XML for this operation?

If those are answered, the CLI command sequence is usually trivial.

---

## 19. PDF-to-Concept Mapping (How to Use the Source Documents)

This section maps **each official PDF** to the concepts and responsibilities it covers, so a new agent knows exactly **where to look** when answering a specific type of question.

---

### 1. *Command Line Interface - RealityScan Help.pdf*

**Primary role:**

* Entry point and mental model of the CLI

**Covers:**

* How CLI commands are structured
* How commands are passed to the executable
* How `.rscmd/.rccmd` files are executed
* Sequential execution model

**Use this PDF when:**

* Explaining how the CLI fundamentally works
* Answering "can the CLI do X at all?"
* Clarifying syntax rules and invocation patterns

---

### 2. *List of All CLI Commands - RealityScan Help.pdf*

**Primary role:**

* Master index of available commands

**Covers:**

* Every CLI command name
* Short descriptions of each command

**Use this PDF when:**

* Verifying whether a command exists
* Finding the exact command spelling
* Discovering adjacent or related commands

**Not sufficient by itself** for correct usage - must be paired with domain PDFs below.

---

### 3. *Settings' Commands - RealityScan Help.pdf*

**Primary role:**

* Configuration control

**Covers:**

* `-set` vs `-preset`
* Runtime vs pre-processing settings
* Coordinate systems
* Reset behaviors

**Use this PDF when:**

* A command "does nothing" due to wrong setting scope
* Translating GUI presets into CLI
* Tuning alignment or reconstruction quality

---

### 4. *Alignment Commands - RealityScan Help.pdf*

**Primary role:**

* Camera alignment and component generation

**Covers:**

* Feature detection
* Alignment execution
* Component creation and selection
* Control points and camera grouping

**Use this PDF when:**

* Working with camera poses
* Exporting registrations
* Debugging multi-component results
* Preparing data for COLMAP / Nerfstudio / SfM pipelines

---

### 5. *Reconstruction Commands - RealityScan Help.pdf*

**Primary role:**

* Meshing and reconstruction

**Covers:**

* Reconstruction region tools
* Preview / Normal / High model computation
* Resume and continuation

**Use this PDF when:**

* Automating mesh generation
* Explaining why a model is empty or clipped
* Adjusting reconstruction fidelity

---

### 6. *Model Tools - RealityScan Help.pdf*

**Primary role:**

* Post-processing meshes

**Covers:**

* Simplification
* Smoothing
* Cutting
* UV unwrap
* Texturing
* Model export

**Use this PDF when:**

* Automating mesh cleanup
* Preparing assets for downstream engines
* Exporting final geometry

---

### 7. *Error-handling Commands - RealityScan Help.pdf*

**Primary role:**

* Robust automation and headless operation

**Covers:**

* Silent mode
* Crash handling
* Progress logging
* Error-triggered termination

**Use this PDF when:**

* Running unattended jobs
* Building batch pipelines
* Integrating with schedulers or CI systems

---

### 8. *Delegation of Commands - RealityScan Help.pdf*

**Primary role:**

* Multi-instance orchestration

**Covers:**

* Instance naming
* Delegating commands
* Waiting, polling, aborting

**Use this PDF when:**

* Running multiple projects in parallel
* Managing long-running jobs
* Building farm-like execution

---

### 9. *Commands Outside Command Prompt - RealityScan Help.pdf*

**Primary role:**

* OS-level integration edge cases

**Covers:**

* Commands triggered outside standard CLI usage
* Drag-and-drop behaviors

**Use this PDF when:**

* Reconciling GUI-only behaviors
* Explaining why some actions are not scriptable

---

## 20. How a New Agent Should Navigate the PDFs

If the task is:

* **"Does RC/RS support this?"** -> *List of All CLI Commands*
* **"Why does this setting not apply?"** -> *Settings' Commands*
* **"Why is alignment/export wrong?"** -> *Alignment Commands*
* **"Why is the mesh empty/bad?"** -> *Reconstruction Commands*
* **"How do I post-process/export models?"** -> *Model Tools*
* **"How do I make this robust/headless?"** -> *Error-handling Commands*
* **"How do I run multiple jobs?"** -> *Delegation of Commands*
* **"What is the CLI actually doing?"** -> *Command Line Interface*

---

## Official CLI Help (Merged HTML Source)

# RealityScan CLI (HTML Source)

## Alignment Commands - RealityScan Help

# Alignment Commands

The following commands can be used to calculate camera poses and calibration, and to handle components and control points.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| align |  |  | Align images using the current settings. |
| draft |  |  | Align images in the draft mode using the current settings. |
| update |  |  | Update all components and models by a rigid transformation to fit actual constraints and control points. |
| detectFeatures |  |  | Run feature detection according to the alignment settings. Detected features will be saved in the application cache. |
| mergeComponents |  |  | Merge already created components. When using this command, no new images are added to existing components. |
| exportXMP |  | params.xml | Export camera metadata of components created in the last alignment in XMP format using the current settings or the settings from params.xml file (optional parameter). You can export this file from the `XMP metadata export` dialog. The components must fulfil the condition defined by the command setMinComponentSize. NOTE: XMP files are stored in the same folder as the respective images. |
| exportXMPForSelectedComponent |  |  | Export camera metadata of a selected component in XMP format using the current settings. An example available in the paragraph 'Metadata (XMP) Export Settings' below. NOTE: XMP files are stored in the same folder as the respective images. |
| importComponent | component.rsalign |  | Import a component from the component.rsalign file. |
| importBundler | filePath | params.xml | Import a Bundler project. The command requires the path to the Bundler file and, optionally, a configuration file that defines the scene transformation settings saved from the import dialog. The configuration file can be used to adjust the coordinate system or apply custom transformations during import. |
| importColmap | filePath | params.xml | Import a COLMAP project. The command requires the path to the COLMAP file (any of the three text files) and, optionally, a configuration file that defines the scene transformation settings saved from the import dialog. The configuration file can be used to adjust the coordinate system or apply custom transformations during import. |
| exportLatestComponents | folderName |  | Export components created in the last alignment as RealityScan Alignment Components (.rsalign) into a specified folder. The components must fulfil the condition defined by the command setMinComponentSize. |
| setMinComponentSize | size |  | Specify the minimal component size for export when using the exportLatestComponents and exportXMP commands. The default value is 5. |
| exportSelectedComponentDir | folderName |  | Export the selected component into a folder (folderName including the path) as a RealityScan Alignment Component (.rsalign). |
| exportSelectedComponentFile | fileName |  | Export the selected component into a RealityScan Alignment Component file. |
| exportRegistration | fileName | params.xml | Export registration to a specified file using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Registration` dialog. |
| exportUndistoredImages | folderName | params.xml | Export undistorted images to a specified folder using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Registration` dialog. |
| exportSTMap |  | folderName params.xml | Export ST maps for the selected images. If folderName and params.xml are not specified, results are stored along with the original images using the current settings. You can export the settings file from the application in the `Export Registration` dialog. |
| exportSparsePointCloud | fileName | params.xml | Export 3D tie points (a sparse point cloud) into a specified file (fileName including the path and format extension) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application within the `Export Point Cloud` dialog. |
| selectComponent | componentName |  | Select a component with the specified name (componentName) for further processing. |
| selectMaximalComponent |  |  | Select the largest component for further processing. |
| selectComponentWithLeastReprojectionError |  |  | Select the component with the smallest reprojection error, based on the calculated Mean error [pixels], for further processing. |
| renameSelectedComponent | newComponentName |  | Rename the currently selected component. |
| deleteSelectedComponent |  |  | Delete the currently selected component. |
| deleteComponent | index |  | Delete a component based on the chosen index. Keep in mind that index numbers start at 0 (zero). |
| deleteAllComponents |  |  | Delete all components. |
| importFlightLog | flFileName | params.xml | Import a trajectory file using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Import Flight Log` dialog. |
| importGroundControlPoints | gcpFileName | params.xml | Import ground control points using settings from the params.xml file. You can export these settings from the application in the `Import Ground Control Points` dialog. |
| importControlPointsMeasurements | cpmFileName | params.xml | Import measurements of control points (CPs) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Import Control Points Measurements` dialog. |
| editControlPointSelection | "key=value" |  | Edit the settings/parameters of the selected control points based on the value in the `Selected control point(s)` panel or its key. More information can be found here . |
| listControlPoints | fileName |  | Export a list of control points to the specified file path, including the file name and extension (fileName paramater). Each control point will be listed with its index. |
| selectControlPoint | controlPointName |  | Select a control point by its name. |
| invertControlPointSelection | controlPointName |  | Invert the control point selection. If no control points are selected, all will be selected. |
| renameControlPoint | controlPointName newName |  | Rename a control point. Specify control point by its current name (controlPointName) and the new name you want to assign (newName). |
| renameSelectedControlPoint | newName |  | Rename the selected control point to the specified new name (newName). |
| deleteControlPoint |  | index | Delete a selected control point. If an optional parameter index is used, a control point with a corresponding index will be removed. Index numbers start at 0 (zero) and follow the order of control points in the 1Ds view. |
| selectMeasurementByError | errorValue | controlPointName | Select any measurement with a position error (in pixels) equal to or greater than the specified errorValue (float value). If no control point is specified (controlPointName), the selection applies to all measurements regardless of their assigned control point. |
| selectMeasurementByIndex | controlPointName index |  | Select a control point measurement by its index within the specified control point. |
| deleteControlPointMeasurement |  |  | Remove selected control points measurements (images assigned to a control point). Images have to be selected in the 1Ds view under the corresponding control point. |
| exportGroundControlPoints | gcpFileName | params.xml | Export ground control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Ground Control` dialog. |
| exportControlPointsMeasurements | cpmFileName | params.xml | Export measurements of control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Control Points Measurements` dialog. |
| defineDistance | PointNameA PointNameB distance | constraintName | Define a distance constraint between two control points. If the constraintName is not defined, the distance name is created automatically. Alternatively, you can load a distance constraint from a file (see below). |
| defineDistance | fileName | params.xml | Import a distance constraint from a file (filename including the path) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Import Distance Definitions` dialog. The list of supported formats distancedefinitions.xml can be found in the installation folder. |
| editConstraintSelection | "key=value" |  | Edit the settings/parameters of the selected constraints based on the value in the `Selected constraint(s)` panel or its key. More information can be found here . |
| deleteConstraint |  | index | Remove the selected distance constraints. The constraints must be selected in the 1Ds view. You can use an index as an optional parameter to specify which constraint to remove. Indexes start at 0 and follow the order of constraints in the 1Ds view. |
| detectMarkers |  | params.xml | Detect markers in images using the current settings or the settings from the params.xml file (optional parameter). You can export these settings using the `Detect Markers` tool in the application. |
| setCamerasGravityDirection |  | componentID | If the image's XMP file contains gravity information (xcr:Gravity), then the component will be rotated so the -z vector is in the direction of the gravity vector. This does not apply to the dense point cloud (mesh/model), only to the sparse point cloud (alignment). Will apply to the selected component, or to the component defined with the optional parameters (component ID). |

## Alignment Examples

Follow these examples to learn how to align images with the command line.

### Alignment and draft

Supposing you want to add images, make fast draft alignment, look at it and then continue to normal alignment and other calculations.

```
RealityScan.exe -addFolder %MyPath%\Images\ -save %MyPath%\MyProject.rsproj -draft
RealityScan.exe -load %MyPath%\MyProject.rsproj -align .... -quit
```

The first line does not end with quit, so the command will wait until the application is closed and continue just after that.

### Export XMP to reuse alignment

Now we will reuse the alignment. Imagine we have a properly aligned project `Project1.rsproj` that uses images from a folder `Images1` and we want to create a project `Project2.rsproj` with images from `Images2` and use the same alignment. We suppose that the corresponding images from `Images1` and `Images2` have the same names, but may differ in their extension.

```
RealityScan.exe -load %MyPath%\Project1.rsproj -exportXMP -quit
copy Images1/*.xmp Images2/
RealityScan.exe -addFolder %MyPath%\Images2\ -align -save %MyPath%\Project2.rsproj -quit
```

The script exports .xmp files from Project1 next to the images (into `Images1` folder), then copies them into `Images2` . The align command will now use the camera positions and calibration according to the setting stored in these .xmp files and application settings.

### Select and export the largest component

The next example takes the largest component and loads it into an empty scene.

```
RealityScan.exe -load %MyPath%\AlignedProject.rsproj -selectMaximalComponent -exportComponent %MyPath%\max -quit
for %%s in ("maxComponent *.rsalign") do (
RealityScan.exe -importComponent "%MyPath%\%%s" -save %MyPath%\NewProject.rsproj -quit
)
```

The name of the component is not known beforehand, therefore it has to be searched for outside the application. Since the names of components are by default called Component 0, Component 1, etc., the largest component will be saved as a `maxComponent 1.rsalign` or likewise.

### Export components above the minimal size

The last example exports all components containing more than 5 cameras and then imports them one by one into a new project.

```
RealityScan.exe -load %MyPath%\AlignedProject.rsproj -minComponentSize 5 -exportComponent %MyPath%\ -newScene -save %MyPath%\NewProject.rsproj -quit
for %%s in (*.rsalign) do (
RealityScan.exe -load %MyPath%\NewProject.rsproj -importComponent "%MyPath%\%%s" -save %MyPath%\NewProject.rsproj -quit
)
```

You can import every component to its own project and process them separately, as well.

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Commands Outside Command Prompt

Using CLI with an .rscmd file

## Delegation of Commands

On delegating commands into an opened instance of RealityScan

## Continue

Model calculation via the command line

## Model Tools' Commands

Further model processing via the command line

## Settings' Commands

Application settings and behaviour

## Error Handling Commands

Commands for handling potential errors

### See also:

- Learn how to reuse alignment click here
- Learn about alignment settings click here
- Learn how to import ground control points click here
- Learn about exporting and joining of components click here
- See all CLI commands in one page click here

## Command Line Interface - RealityScan Help

# Command Line Interface

Many features of RealityScan can be used directly via the command line or by running a batch file. They are passed to the application as parameters of RealityScan.exe and executed in a sequence. The process behaves in a similar way as if the features were called using the GUI. The application will launch and you can interact with it as usual during the calculations.

You can run a command sequence in the Windows Command Prompt or by running a file with .bat extension. This is an example of the command sequence. It starts by launching the application with its full path. Every command begins with a hyphen and it can be follower by parameters.

```
"C:\Program Files\Epic Games\RealityScan\RealityScan.exe" -load C:\MyFolder\MyProject.rsproj -selectMaximalComponent -calculateNormalModel -simplify 1000000 -save C:\MyFolder\MyProject.rsproj -quit
```

To replace the full application path with a simple `RealityScan.exe` command, store the path in a system variable path.

Add the following line to the beginning of your script in order to temporarily add the RealityScan folder to the path (until the end of the command-line session).

```
set PATH=%PATH%;C:\Program Files\Epic Games\RealityScan\
RealityScan.exe -load ... -quit
```

## Basic Project and Image Commands

The following are the most basic commands that let you control the current project, the application itself, and add images.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| headless |  |  | Hides user interface. Find more about this command here . |
| hideUI |  |  | Hide user interface. Unlike `headless` , this command doesn't need to be run at startup and does not suppress actions that require user interaction. |
| showUI |  |  | Shows hidden user interface. Unlike `headless` , this command doesn't need to be run at startup. |
| newScene |  |  | Create a new empty scene. |
| load | MyProject.rsproj | recoverAutosave|deleteAutosave | Load an existing project from the MyProject.rsproj file. Use optional parameters to define the action if there is an autosaved file present for this project. Using `recoverAutosave` will open the autosaved project, while `deleteAutosave` will delete the autosaved project and load the original one. More information about Autosave feature here . To set the preference globally for all projects, use `set` command with `appAutoSaveCliHandling` key. Find more information in the section CLI Settings Keys & Values . |
| save |  | MyProject.rsproj | Save the current project to its original location or save as MyProject.rsproj. |
| start |  |  | Run the processes configured for the `Start` button. Adjust these settings in the `Start button` settings which can be accesed from the `Start` button dropdown menu. |
| unlockPPIProject | myProject.rcproj |  | Save and unlock your PPI projects to make them compatible for use with the latest RealityScan versions. Use the whole path with the project's name and extension where you want to store the unlocked project. |
| add | imageName |  | Import one or more images from a specified file path or from an image list. The image list is a text file with the .imagelist extension, containing full paths to the images, each on a separate line. |
| addFolder | folderName |  | Add all images in the specified folder. In order to include also subdirectories, use command `set` with a key `appIncSubdirs` as follows: `-set "appIncSubdirs=true"` . Find more information in the section CLI Settings Keys & Values . |
| importVideo | videoFileName extractedVideoFramesLocation jumpsLength |  | Import frames extracted from a video (videoFileName including the path). The frames are extracted into a folder (extractedVideoFramesLocation) using an interval between frames defined by the jumpsLength (in seconds). |
| importLeicaBlk3D | fileName |  | Import an image sequence with .cmi extension (fileName including path) captured by Leica BLK3D. |
| importLaserScan | laserScanName | params.xml | Add a LiDAR scan or a LiDAR-scan list using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `LiDAR Scan Import` dialog. |
| importLaserScanFolder | folderName | params.xml | Add all LiDAR scans in the specified folder using settings from the params.xml file. You can export these settings from the application in the `LiDAR Scan Import` dialog. |
| importHDRimages | fileName|folderName|imageList | params.xml | Import HDR image (imageName), list of images (imageList) or all images from a folder (folderName) using the current settings or the settings from params.xml. You can export these settings from the application in the `16-bit/HDR Images Import` dialog. |
| addImageWithCalibration | fileName xmpFileName |  | Import an image as well as the corresponding XMP file. Use whole paths to the files. |
| importImageSelection | fileName |  | Select scene images and/or LiDAR scans listed in a file (fileName including the path). |
| selectImage | imagePath|regexp | set union sub intersect toggle | Select a specified image (imagePath) or images defined by regular expression (regexp). See below examples . One of the optional parameters may be used to more select images with more flexibility. |
| selectAllImages |  |  | Select all images in the project. |
| deselectAllImages |  |  | Deselect all images in the project. |
| invertImageSelection |  |  | Invert the current image selection. |
| removeCalibrationGroups |  |  | Clear all inputs from their calibration groups. |
| generateAIMasks |  |  | Use AI Masking to generate masks by isolating the object of interest in your images. |
| exportMasks | folderPath params.xml |  | Export the mask images currently used in the project. Specify the output folder and, optionally, the full path to the XML file with export parameters (this file can be exported from the `Export Mask Images` dialog). |
| setImageLayer | index pathImage layerType |  | Set the layer from the image defined with the pathImage parameter (path to the layer image) to the image defined with the index parameter. Index corresponds to the image order in the 1Ds view and it starts at 0 (zero). The layerType parameter defines which layer to set onto the chosen image (e.g. mask, texture). |
| setImagesLayer | pathImage layerType |  | Set the layer from the image defined with the pathImage parameter (path to the layer image) to the selection of images. The layerType parameter defines which layer to set onto the selected images (e.g. mask, texture). |
| removeImageLayer | layerType |  | Remove the layers corresponding to the layerType parameter (e.g. mask, texture) from the selected images. |
| importCache | folderName |  | Import resource cache data from the specified folder. |
| clearCache |  |  | Clear the application cache. You must save the project before clearing the application cache. |
| execRSCMD | Commands.rscmd |  | Execute commands from an .rscmd (or .rccmd file), optionally using report parameters. The file format and syntax are described in the Commands Outside Command Prompt section. You can define up to nine arguments, acting as variables, by listing them after the path to the .rscmd file. Inside the file, reference these variables using $(arg1)–$(arg9). |
| quit |  |  | Quit the application. |

### Examples of image selection

Select an image specified by direct path:

```
-selectImage D:\sample\Images\IMG_0018.JPG
```

Select images with "DSC" in their name:

```
-selectImage g/DSC/
```

Select images that contain "2" and "1" and some other character in between (e.g. image_1251.jpg):

```
-selectImage g/2.1/
```

Select every image that ends with even number and has .jpg extension (e.g. 222.jpg or img_0018.jpg):

```
-selectImage g/[02468]\.jpg/
```

Select every image that contains "DSC", ends with even number and has .jpg extension (e.g. img_DSC_222.jpg or DSC_14.jpg):

```
-selectImage g/DSC.*[02468]\.jpg/
```

Command used to select image(s) can have one of the few optional parameters:

- `set` - select only those images which meet the chosen condition and deselect all others. This operation is used by default even if none of the optional parameters is being used.
- `union` - create a union with already selected images. This will not deselect already selected images, but will only add images that meet the chosen conditions to the selection.
- `sub` - deselect images that are already selected and that correspond to the chosen condition.
- `intersect` - select only those images that are already selected and that correspond to the chosen condition. All others will be removed from the selection.
- `toggle` - invert the selection status of images that meet the chosen condition. Already selected images will be deselected, and vice versa.

## Delegate commands

You can use these commands to delegate actions to a certain RealityScan instance. You can find out more about delegation of commands here .

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| setInstanceName | instanceName |  | Assign a name to a RealityScan instance. |
| delegateTo | instanceName|* |  | Delegate a command or a sequence of commands to a specific instance of RealityScan (instanceName) or to the first available instance (using the * symbol as a parameter). |
| waitCompleted | instanceName|* |  | Pause execution of other commands until the current process is finished in a specified instance of RealityScan (instanceName) or in the first available instance (using the * as a parameter). |
| getStatus | instanceName|* |  | Return the progress status of a running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). |
| pauseInstance | instanceName|* |  | Pause a currently running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). |
| unpauseInstance | instanceName|* |  | Unpause a currently paused process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). |
| abortInstance | instanceName|* |  | Abort a currently running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). If processing is done using the CLI commands, all processes after the running process will also be aborted. |
| execRSCMDIndirect | instanceName|* commands.rscmd |  | Execute commands listed in the .rscmd (or .rccmd) file in the specified instance. |

## Commands for Selected Images

The following enable you to set some preferences for further image processing.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| setFeatureSource | 0|1|2 |  | Define a feature source mode for the selected images: 0 - Merge using overlaps, 1 - Use component features, 2 - Use all image features. |
| enableAlignment | true|false |  | Enable/disable selected images in the registration process. |
| enableMeshing | true|false |  | Enable/disable selected images in the model computation/meshing. |
| enableTexturingAndColoring | true|false |  | Enable/disable selected images during the coloring and texture calculation. |
| setWeightInTexturing | <0,1> |  | Set weight for selected images during the coloring and texture calculation. |
| enableColorNormalizationReference | true|false |  | Set selected images as color references in the color normalization process. The colors for these images will not be changed. |
| enableColorNormalization | true|false |  | Enable or disable selected images in the color normalization process. |
| setDownscaleForDepthMaps | integer |  | Set a downscale factor for depth-map computation for the selected images. |
| enableInComponent | true|false |  | Enable selected images in meshing and continue. Applicable only for the registered images. |
| setCalibrationGroupByExif |  |  | Set the calibration group of all inputs based on their Exif. |
| setConstantCalibrationGroups |  |  | Group all selected inputs into a single calibration group. |
| lockPoseForContinue | true|false |  | Set relative camera pose unchanged for the selected images during the next registration. Applicable only for the registered images. |
| setPriorCalibrationGroup | number |  | Set a prior calibration group for the selected images: -1 - do not group, another number - group into the same calibration group. |
| setPriorLensGroup | number |  | Set a prior lens group for the selected images. Using -1 means do not group, any other number means to group the selected images into the same distortion group. |
| editInputSelection | "key=value" |  | Edit the settings of the selected inputs based on the value in the `Selected inputs` panel or its key. More information can be found here . |

## Code Examples

All the scripts are written in Windows command-line language. Let us say that you write a batch script stored in a `C:\MyFolder` and you want to load a project `C:\MyFolder\PlainProject.rsproj` , add images from the folder `C:\MyFolder\Images` , save progress to `MyProject.rsproj` and quit.

```
set PATH=%PATH%;C:\Program Files\Epic Games\RealityScan\
RealityScan.exe -load C:\MyFolder\PlainProject.rsproj -addFolder C:\MyFolder\Images\ -save C:\MyFolder\MyProject.rsproj -quit
```

To improve the legibility (in case the path is long) and flexibility of your code, we recommend you to store the working folder in an environment variable.

```
set PATH=%PATH%;C:\Program Files\Epic Games\RealityScan\
set MyPath=C:\MyFolder
RealityScan.exe -load %MyPath%\PlainProject.rsproj -addFolder %MyPath%\Images\ -save %MyPath%\MyProject.rsproj -quit
```

Adding an image list or a single image works in the same manner.

```
set PATH=%PATH%;C:\Program Files\Epic Games\RealityScan\
set MyPath=C:\MyFolder
RealityScan.exe -load %MyPath%\PlainProject.rsproj -add %MyPath%\images.imagelist -add %MyPath%\Images\image_123.jpg -save %MyPath%\MyProject.rsproj -quit
```

Using the `-newScene` command is usually not necessary, since the application launches with a new scene. But it can be useful to start a new project without a need to relaunch the application.

In batch scripting, you always need to break a line of code with `^` in case you want to continue the sequence of RealityScan commands on the next line and process them as one. Any lines beginning with `#` , `//` , `REM` or `rem` are skipped. You can use these symbols before a portion of text to mark it as a code comment, for instance: `// I am a comment` .

## Continue

Using CLI with .rscmd

## Delegation of Commands

On delegating commands into an opened instance of RealityScan

## Alignment Commands

Commands for alignment and component handling

## Reconstruction Commands

Model calculation via the command line

## Model Tools' Commands

Further model processing via the command line

## Settings' Commands

Application settings and behaviour

## Error Handling Commands

Commands for handling potential errors

### See also:

- All CLI commands in one page click here

## Commands Outside Command Prompt - RealityScan Help

# Commands Outside Command Prompt

RealityScan enables you to run commands even without the direct use of the command line. You can do so very simply with drag-and-dropping a text file, which includes a sequence of supported commands of your choice, saved with an .rscmd extension, into the application.

## Example Explained

This is an example of one command sequence that may be written inside the .rscmd file, used for loading images stored in the folder `C:\MyFolder\Images\` , aligning all the loaded pictures, creating a model in normal detail, and saving the project to `C:\MyFolder\MyProject.rsproj` :

```
-addFolder C:\MyFolder\Images\ -align -setReconstructionRegionAuto -calculateNormalModel -save C:\MyFolder\MyProject.rsproj
```

You can either write all commands in a row (like above) or write each one in a new/separate line, which helps making the code better visually structured, like this:

```
-addFolder C:\MyFolder\Images\
-align
-setReconstructionRegionAuto
-calculateNormalModel
-save C:\MyFolder\MyProject.rsproj
```

You may also use the `^` sign to break a line (with or without a space before `^` ):

```
-addFolder C:\MyFolder\Images\^
-align^
-setReconstructionRegionAuto^
-calculateNormalModel^
-save C:\MyFolder\MyProject.rsproj
```

## Executing commands listed in .rscmd file

You can use command `execrscmd` to execute all commands listed in the specified .rscmd file. Required parameter for `execrscmd` command is the full file and pathname for the .rscmd file. You can use `execrscmd` command in the Windows command line, in your .bat file or inside another .rscmd file (see example below).

## Passing arguments into .rscmd file

When using a command `execrscmd` to execute the .rscmd file, you can pass up to 10 arguments - the first argument and path to the executed file is required. In order to use the arguments inside the .rscmd file, use $(arg0) $(arg1) ... $(arg9). Below is an example of using sample.bat file to open RealityScan and calling addFolder.rscmd that adds images from the folder "D:\MyFolder\Images" into the application. "D:\MyFolder\Images" is the only argument passed to addFolder.rscmd. File sample.bat file contains:

```
"C:\Program Files\Epic Games\RealityScan\RealityScan.exe" -execrscmd D:\MyFolder\addFolder.rscmd "D:\MyFolder\Images"
```

File addFolder.rscmd contains:

```
-addFolder $(arg1)
```

## Using RealityScan functions and variables

When using RealityScan command or executing a .rscmd file, you can use basic predefined RealityScan functions and variables. The same functions and variables can be used when exporting "Reports". You can find the instructions on how to use them as well as the list of available functions and variables, here in the section Reports - Basic functions. Below you can find the commands inside the .rscmd . Once you drag and drop the .rscmd into the application, the following actions are performed:

- A new scene is opened.
- All inputs from folder D:\MyFolder\1 are added into the application.
- Inputs from the folder are aligned and a preview model is created with an automatic reconstruction region.
- Project is saved into D:\MyFolder\project_1.rsproj
- A new scene is opened.
- All inputs from folder D:\MyFolder\2 are added into the application.
- Inputs from the folder are aligned and a preview model is created with an automatic reconstruction region.
- Project is saved into D:\MyFolder\project_2.rsproj

Example:

```
$For( "i", 1, 1, 3,
-newScene
-addFolder D:\MyFolder\$(i)
-align
-setReconstructionRegionAuto
-calculatePreviewModel
-save D:\MyFolder\project_$(i).rsproj
)
```

Additionally, you can use these global variables:

$(appRootDir) - path to the installation folder of the RealityScan application. Most often it is C:\Program Files\Epic Games\RealityScan.

$(appStartDir) - path to the folder from which the RealityScan.exe was executed (e.g. the path to the .bat file that calls RealityScan.exe).

$(cmdStartDir) - path to the folder in which the executed .rscmd file is located.

$(arg1), ..., $(arg9) - variables passed to execrscmd command.

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Continue

On delegating commands into an opened instance of RealityScan

## Alignment Commands

Commands for alignment and component handling

## Reconstruction Commands

Model calculation via the command line

## Model Tools' Commands

Further model processing via the command line

## Settings' Commands

Application settings and behaviour

## Error Handling Commands

Commands for handling potential errors

### See also:

- See all CLI commands in one page click here

## Delegation of Commands - RealityScan Help

# Delegation of Commands

This tutorial will introduce you to naming of running instances of RealityScan, delegating commands into one of them, pausing, aborting and checking processes via the command-line interface.

## Instance Naming

RealityScan allows you to open up to 4 instances at once. This means that you can have 4 projects open in RealityScan at the same time. When using the command-line interface, you can name individual instances and then delegate commands into an already opened instance – with the use of the command `setInstanceName` with a parameter `instanceName` . The name of the instance cannot contain spaces. EXAMPLE - the following will set the name of the instance to RS1:

```
RealityScan.exe -setInstanceName RS1
```

## Delegation of Commands

You can delegate a command or a sequence of commands to an already opened instance of RealityScan using the command `delegateTo` with these 2 parameters: `instanceName` and `commandsDefinition` . Instead of `instanceName` , you can use the star symbol (*). In this case, the command is delegated to the first instance found. EXAMPLE 1 - to add an image to the first, already opened instance of RealityScan, write:

```
RealityScan.exe -delegateTo * -add D:\\datasets\\test\\img001.JPG
```

EXAMPLE 2 - for adding an image to the instance of RealityScan named RS1, use:

```
RealityScan.exe -delegateTo RS1 -add D:\\datasets\\test\\img001.JPG
```

EXAMPLE 3 - this adds every fifth image from a specified folder, aligns the images, saves the project and quits the app:

```
start RealityScan.exe
TIMEOUT 2
for /l %%A in (1,5,100) do (
RealityScan.exe -delegateTo * -add D:\\datasets\\test\\img%%A.JPG
)
RealityScan.exe -delegateTo * -align
RealityScan.exe -delegateTo * -save d:\\datasets\\test\\processed\\scene.rsproj
RealityScan.exe -delegateTo * -quit
```

## Waiting for Completion

There is also the command `waitCompleted` for pausing the execution of other commands until the current process is finished. When using this command along with the parameter `instanceName` (or the * symbol), the following commands are executed only once the process is finished in the instance that the command is referring to. See the example below. EXAMPLE - run alignment in the first instance found, wait with the batch execution until the alignment process is finished and then check the result of the process:

```
...
RealityScan.exe -delegateTo * -align
RealityScan.exe -waitCompleted *
call CheckResult
...
```

## Checking Instance Status

Use the command `getStatus` along with `instanceName` (or an asterisk "*") to return the progress of a running instance. EXAMPLE - retrieves the current status from an already opened instance RS1:

```
RealityScan.exe -getStatus RS1
```

RESULT - in the console of that specific instance, you will get the result in the form of the progress ID, progress percentage of the current process, elapsed time, and estimated time:

```
id:0x10001 progress:57.5% runtime:4.26sec endEstimation:3.40sec
```

You can also print the result of `getStatus` to an external text file. The command would look as such:

```
RealityScan.exe -getStatus * > D:\statusreport.txt
```

## Controlling Process

You can use commands `pauseInstance` , `unpauseInstance` , and `abortInstance` along with the parameter `instanceName` (or a * symbol) to control the currently running processes inside of an opened instance of RealityScan. This is beneficial mainly when processing on servers or render farms. These commands allow you to pause a current process in case there is a need to run the process with a higher priority. Afterwards, you can again unpause the previous process, when needed. Alternatively, you can abort the process completely. EXAMPLE - pause a process running in the instance RS1:

```
RealityScan.exe -pauseInstance RS1
```

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Commands Outside Command Prompt

Using CLI with an .rscmd file

## Continue

Commands for alignment and component handling

## Reconstruction Commands

Model calculation via the command line

## Model Tools' Commands

Further model processing via the command line

## Settings' Commands

Application settings and behaviour

## Error Handling Commands

Commands for handling potential errors

### See also:

- See all CLI commands in one page click here

## Error-handling Commands - RealityScan Help

# Error-handling Commands

With the following commands and settings you can handle potential errors occurring during the application command-line processing.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| silent | crashReportPath |  | Suppress warning dialogs and crash reports uploads. The application will store reports to the specified location instead of showing the upload wizard. |
| set | "key=value" |  | Set an application state variable. |
| preset | "key=value" |  | Change an application setting during the setup phase. Ideal for changes that require a reset of the application. Learn more about how to use this command here . |
| reset | ui | cfg | cfgui | all |  | Reset the user interface, settings, or both. It is also possible to make RealityScan like a clean install. This command works only when used in a batch file, and it won't work with delegation commands. What is going to be reset is determined by the chosen parameter: ui - reset user interface, cfg - reset application settings, cfgui - reset both interface and settings, all - make it like a clean install. |
| writeProgress | fileName | timeout | Write a progress change into a specified file (fileName including the path) during a defined period of time (timeout in seconds). The file structure in the examples below. You can see all process IDs here . |
| printProgress |  | timeout | Print a progress change into the Windows Command Prompt for any new change. Optional timeout parameter will also output during a defined period of time (timeout in seconds). |
| tag |  |  | Writes out a tag into the Windows Command Prompt. It respects the order of the used commands. That means that it will be shown only after the process that was run with the command used before is finished. |
| stdConsole |  |  | Enables console redirection to the application standard output. When used, you will see the application console content mirrored also in the standard Windows console. This also enables further redirections for CLI purposes. |

| Setting Name | Key | Value | Default value |
| --- | --- | --- | --- |
| Quit on error | appQuitOnError | bool | false |
| Quit on required restart | appQuitOnReset | bool | false |
| Suppress error messages | suppressErrors | bool | false |
| Minimal process duration | appProcessActionTime | int | 15 |
| Action | appProcessAction | None | None |
| PlaySound |  |  |  |
| ExecuteProgram |  |  |  |
| Command-line process* *relevant for: "appProcessAction=ExecuteProgram" | appProcessExecCmd | string |  |

## Examples and More Info

### Silent Crash Report

```
RealityScan.exe -silent c:\\CrashReportFolder ^
-set "appQuitOnError=true" ^
-set "appProcessActionTime=0" ^
-set "appProcessAction=ExecuteProgram" ^
-set "appProcessExecCmd=c:\\MyScripts\\ErrorWriter.bat $(processResult) $(processId) $(processDuration:d) c:\\ErrorReportFolder\\ErrorReport.txt"
```

The -silent command with the parameter c:\\CrashReportFolder redirects crash reporting minidumps to the target folder CrashReportFolder .

Setting appQuitOnError to True causes the application to quit if any error occurs.

appProcessActionTime set to 0 means that we are interested in all processes whose duration is at least 0 seconds - so we are actually interested in every single process. Setting appProcessAction to 2 stands for setting it to Execute a program . appProcessExecCmd is set to a specific command (batch script) ErrorWriter.bat with its full path c:\\MyScripts\\ErrorWriter.bat and necessary input parameters $(processResult) $(processId) $(processDuration:d) c:\\ErrorReportFolder\\ErrorReport.txt . These three settings can be interpreted together in such a way that if process duration has taken more than the minimum specified time (0 seconds), then the application executes the program ErrorWriter.bat .

You can use values from the application as input parameters for the executed program, e.g.: processResult - a number which represents a result of the process (the process has finished correctly if its processResult is 0), processId - an ID of the process, processDuration:d - a number which says how long it has taken to finish the process. You can also use your own input parameters (e.g. c:\\ErrorReportFolder\\ErrorReport.txt ) to send some necessary data into the script, like a name and full path to a text file where a report of discovered errors is written.

Here is a simple example of a short batch script ErrorWriter.bat :

```
if /i "%1" NEQ "0" ( 
                if /i "%1" NEQ "1" ( 
                    echo An error occured by process %2 which finished with result code %1 in %3 seconds. > %4 
                ) 
)
```

where %1, %2, %3, %4 are input parameters $(processResult), $(processId), $(processDuration:d), c:\\ErrorReportFolder\\ErrorReport.txt . This script checks the value of the input parameter processResult. If the value is not 0, the process has finished with an error and the script writes a short report to the output ErrorReport text file:

```
An error occurred by process 20599 which finished with result code 2181038335 in 0 seconds.
```

### Exit code

In case RealityScan process finished successfully, the application returns exit code equal to 0 (you can see the result in Windows command prompt).

If the RealityScan process finishes with error, the application returns decimal code of that specific error (in case of using a command -set "appQuitOnError=true" ).

If the RealityScan process crashes with minidump, the application returns exit code equal to 3.

Example of the code:

```
RealityScan.exe -silent c:\\CrashReportFolder ^
-set "appQuitOnError=true"
…
echo RealityScan.exe returned with %errorlevel%.
```

Example of the result in case of red error:

Example of the result in case of crash with minidump:

### Quit on Restart

There are some application settings that require application restart after changing them. In such cases, you can use the -set command with the key appQuitOnReset set to True to suppress the dialog. Please note that the application quits after using this key and changing the respective setting. Below you can find an example on how to change the cache directory to a custom location, and then open a new scene:

```
RealityScan.exe -set "appQuitOnReset=true" ^
-set "appCacheLocation=Custom"
RealityScan.exe -set "appQuitOnReset=true" ^
-set "appCacheCustomLocation=D:\cr-tmp"
RealityScan.exe -newScene
```

### Progress Monitoring

With the -writeProgress command, you are able to write a progress information into a specified file (fileName) during a defined period of time (timeout). The created file consists of these 5 columns: algId – process ID, progress – number from this interval <0,1> indicating a stage of a process, duration – elapsed time in seconds, estimation – estimated remaining time in seconds, eventType – {started, progress, timeout, completed}, an event that produced a specific record.

```
20561 0.00 0.04 404.08 #started
20561 0.45 0.10 0.22 #progress
20561 0.85 14.76 3.26 #progress
20561 0.90 15.04 1.70 #progress
20561 0.91 16.18 1.76 #progress
20561 0.94 16.72 1.82 #progress
20561 0.96 16.78 1.42 #progress
20561 0.97 17.06 0.55 #progress
20561 1.00 17.10 0.10 #progress
20561 1.00 17.13 0.00 #completed
```

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Commands Outside Command Prompt

Using CLI with an .rscmd file

## Delegation of Commands

On delegating commands into an opened instance of RealityScan

## Alignment Commands

Commands for alignment and component handling

## Reconstruction Commands

Model calculation via the command line

## Model Tools' Commands

Further model processing via the command line

## Settings' Commands

Application settings and behaviour

## Process IDs

A list of the process IDs

### See also:

- See all CLI commands in one page click here
- See the application settings' keys and values click here

## List of All CLI Commands - RealityScan Help

# List of All CLI Commands

Use CLI commands to automate your workflow and make iterative processing easier.

## Project and Images

Add images to your project or change the project/instance settings.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| headless |  |  | Hides user interface. Find more about this command here . |
| hideUI |  |  | Hide user interface. Unlike `headless` , this command doesn't need to be run at startup and does not suppress actions that require user interaction. |
| showUI |  |  | Shows hidden user interface. Unlike `headless` , this command doesn't need to be run at startup. |
| newScene |  |  | Create a new empty scene. |
| load | MyProject.rsproj | recoverAutosave|deleteAutosave | Load an existing project from the MyProject.rsproj file. Use optional parameters to define the action if there is an autosaved file present for this project. Using `recoverAutosave` will open the autosaved project, while `deleteAutosave` will delete the autosaved project and load the original one. More information about the Autosave feature can be found here . To set the preference globally for all projects, use `set` command with `appAutoSaveCliHandling` key. Find more information in the section CLI Settings Keys & Values . |
| save |  | MyProject.rsproj | Save the current project to its original location or save as MyProject.rsproj. |
| start |  |  | Run the processes configured for the `Start` button. Adjust these settings in the `Start button` settings which can be accesed from the `Start` button dropdown menu. |
| unlockPPIProject | myProject.rcproj |  | Save and unlock your PPI projects to make them compatible for use with the latest RealityScan versions. Use the whole path with the project's name and extension where you want to store the unlocked project. |
| add | imageName |  | Import one or more images from a specified file path or from an image list. The image list is a text file with the .imagelist extension, containing full paths to the images, each on a separate line. |
| addFolder | folderName |  | Add all images to the specified folder. To include subdirectories, use the command `set` with a key `appIncSubdirs` as follows: `-set "appIncSubdirs=true"` . Find more information in the section CLI Settings Keys & Values . |
| importVideo | videoFileName extractedVideoFramesLocation jumpsLength |  | Import frames extracted from a video (videoFileName including the path). The frames are extracted into a folder (extractedVideoFramesLocation) using an interval between frames defined by the jumpsLength (in seconds). |
| importLeicaBlk3D | fileName |  | Import image sequence with .cmi extension (fileName including path) captured by Leica BLK3D. |
| importLaserScan | laserscanName | params.xml | Add a LiDAR scan or a LiDAR scan list using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `LiDAR Scan Import` dialog. |
| importLaserScanFolder | folderName | params.xml | Add all LiDAR scans in the specified folder using settings from the params.xml file. You can export these settings from the application in the `LiDAR Scan Import` dialog. |
| importHDRimages | fileName|folderName|imageList | params.xml | Import HDR image (imageName), list of images (imageList) or all images from a folder (folderName) using the current settings or the settings from params.xml. You can export these settings from the application in the `16-bit/HDR Images Import` dialog. |
| addImageWithCalibration | fileName xmpFileName |  | Import an image as well as the corresponding XMP file. Use whole paths to the files. |
| importImageSelection | fileName |  | Select some scene images and/or LiDAR scans listed in a file (filename including the path). |
| selectImage | imagePath|regexp | set union sub intersect toggle | Select a specified image (imagePath) or images defined by regular expression (regexp). See below examples . One of the optional parameters may be used to more select images with more flexibility. |
| selectAllImages |  |  | Select all images in the project. |
| deselectAllImages |  |  | Deselect all images in the project. |
| invertImageSelection |  |  | Invert the current image selection. |
| removeCalibrationGroups |  |  | Clear all inputs from their calibration groups. |
| generateAIMasks |  |  | Use AI Masking to generate masks by isolating the object of interest in your images. |
| exportMasks | folderPath params.xml |  | Export the mask images currently used in the project. Specify the output folder and, optionally, the full path to the XML file with export parameters (this file can be exported from the `Export Mask Images` dialog). |
| setImageLayer | index pathImage layerType |  | Set the layer from the image defined with the pathImage parameter (path to the layer image) to the image defined with the index parameter. The index corresponds to the image order in the 1Ds view, starting at 0 (zero). The layerType parameter defines which layer to set onto the chosen image (e.g., mask, texture). |
| setImagesLayer | pathImage layerType |  | Set the layer from the image defined with the pathImage parameter (path to the layer image) to the selection of images. The layerType parameter defines which layer to set onto the selected images (e.g., mask, texture). |
| removeImageLayer | layerType |  | Remove the layers corresponding to the layerType parameter (e.g. mask, texture) from the selected images. |
| importCache | folderName |  | Import resource cache data from a specified folder. |
| clearCache |  |  | Clear the application cache. You must save the project before clearing the application cache. |
| execRSCMD | Commands.rscmd |  | Execute commands from an .rscmd (or .rccmd file), optionally using report parameters. The file format and syntax are described in the Commands Outside Command Prompt section. You can define up to nine arguments, acting as variables, by listing them after the path to the .rscmd file. Inside the file, reference these variables using $(arg1)–$(arg9). |
| quit |  |  | Quit the application. |

### Commands for Selected Images

The following ones enable you to set some preferences for further processing of images.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| setFeatureSource | 0|1|2 |  | Define a feature source mode for the selected images: 0 – Merge using overlaps, 1 – Use component features, 2 – Use all image features. |
| enableAlignment | true|false |  | Enable/disable selected images in the registration process. |
| enableMeshing | true|false |  | Enable/disable selected images in the model computation/meshing. |
| enableTexturingAndColoring | true|false |  | Enable/disable selected images during the coloring and texture calculation. |
| setWeightInTexturing | <0,1> |  | Set weight for selected images during the coloring and texture calculation. |
| enableColorNormalizationReference | true|false |  | Set selected images as color references in the color normalization process. The colors for these images will not be changed. |
| enableColorNormalization | true|false |  | Enable or disable selected images in the color normalization process. |
| setDownscaleForDepthMaps | integer |  | Set a downscale factor for depth-map computation for the selected images. |
| enableInComponent | true|false |  | Enable selected images in meshing and continue. Applicable only for the registered images. |
| setCalibrationGroupByExif |  |  | Set the calibration group of all inputs based on their Exif. |
| setConstantCalibrationGroups |  |  | Group all selected inputs into a single calibration group. |
| lockPoseForContinue | true|false |  | Set relative camera pose unchanged for the selected images during the next registration. Applicable only for the registered images. |
| setPriorCalibrationGroup | number |  | Set a prior calibration group for the selected images: -1 - do not group, another number - group into the same calibration group. |
| setPriorLensGroup | number |  | Set a prior lens group for the selected images. Using -1 means do not group; any other number means to group the selected images into the same distortion group. |
| editInputSelection | "key=value" |  | Edit the settings of the selected inputs based on the value in the `Selected inputs` panel or its key. More information can be found here . |

## Delegate commands

You can use these commands to delegate actions to a certain RealityScan instance. You can find out more about the delegation of commands here .

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| setInstanceName | instanceName |  | Assign a name to a RealityScan instance. |
| delegateTo | instanceName|* |  | Delegate a command or a sequence of commands to a specific instance of RealityScan (instanceName) or to the first available instance (using the * symbol as a parameter). |
| waitCompleted | instanceName|* |  | Pause execution of other commands until the current process is finished in a specified instance of RealityScan (instanceName) or in the first available instance (using the * as a parameter). |
| getStatus | instanceName|* |  | Return the progress status of a running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). |
| pauseInstance | instanceName|* |  | Pause a currently running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). |
| unpauseInstance | instanceName|* |  | Unpause a currently paused process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). |
| abortInstance | instanceName|* |  | Abort a currently running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). If processing is done using the CLI commands, all processes after the running process will also be aborted. |
| execRSCMDIndirect | instanceName|* commands.rscmd |  | Execute commands listed in the .rscmd (or .rccmd) file in the specified instance, optionally using report parameters. The file format and syntax are described in the Commands Outside Command Prompt section. You can define up to nine arguments, acting as variables, by listing them after the path to the .rscmd file. Inside the file, reference these variables using $(arg1)–$(arg9). |

## Alignment

The following commands can be used to calculate camera poses and calibration and to handle components and control points.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| align |  |  | Align images using the current settings. |
| draft |  |  | Align images in the draft mode using the current settings. |
| update |  |  | Update all components and models by a rigid transformation to fit the actual constraints and control points. |
| detectFeatures |  |  | Run feature detection according to the alignment settings. Detected features will be saved in the application cache. |
| mergeComponents |  |  | Merge already created components. When using this command, no new images are added to the existing components. |
| exportXMP |  | params.xml | Export camera metadata of components created in the last alignment in the XMP format using the current settings or the settings from params.xml file (optional parameter). You can export this file from the `XMP metadata export` dialog. The components must fulfill the condition defined by the command setMinComponentSize. NOTE: XMP files are stored in the same folder as the respective images. |
| exportXMPForSelectedComponent |  |  | Export camera metadata of a selected component in XMP format using the current settings. An example is available in the paragraph "Metadata (XMP) Export Settings" below. NOTE: XMP files are stored in the same folder as the respective images. |
| importComponent | component.rsalign |  | Import a component from the component.rsalign file. |
| importBundler | filePath | params.xml | Import a Bundler project. The command requires the path to the Bundler file and, optionally, a configuration file that defines the scene transformation settings saved from the import dialog. The configuration file can be used to adjust the coordinate system or apply custom transformations during import. |
| importColmap | filePath | params.xml | Import a COLMAP project. The command requires the path to the COLMAP file (any of the three text files) and, optionally, a configuration file that defines the scene transformation settings saved from the import dialog. The configuration file can be used to adjust the coordinate system or apply custom transformations during import. |
| exportLatestComponents | folderName |  | Export components created in the last alignment as RealityScan Alignment Components (.rsalign) into a specified folder. The components must fulfill the condition defined by the command setMinComponentSize. |
| setMinComponentSize | size |  | Specify the minimal component size for export when using the exportLatestComponents and exportXMP commands. The default value is 5. |
| exportSelectedComponentDir | folderName |  | Export the selected component into a folder (folderName including the path) as a RealityScan Alignment Component (.rsalign). |
| exportSelectedComponentFile | fileName |  | Export the selected component into a RealityScan Alignment Component file. |
| exportRegistration | fileName | params.xml | Export registration to a specified file using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Registration` dialog. |
| exportUndistortedImages | folderName | params.xml | Export undistorted images into a specified folder using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Registration` dialog. |
| exportSTMap |  | folderName params.xml | Export ST maps for the selected images. If folderName and params.xml are not specified, results are stored along with the original images using the current settings. You can export the settings file from the application in the `Export Registration` dialog. |
| exportSparsePointCloud | fileName | params.xml | Export 3D tie points (a sparse point cloud) into a specified file (fileName including the path and format extension) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application within the `Export Point Cloud` dialog. |
| selectComponent | componentName |  | Select a component with the specified name (componentName) for further processing. |
| selectMaximalComponent |  |  | Select the largest component for further processing. |
| selectComponentWithLeastReprojectionError |  |  | Select the component with the smallest reprojection error, based on the calculated Mean error [pixels], for further processing. |
| renameSelectedComponent | newComponentName |  | Rename the currently selected component. |
| deleteSelectedComponent |  |  | Delete the currently selected component. |
| deleteComponent | index |  | Delete a component based on the chosen index. Keep in mind that index numbers start at 0 (zero). |
| deleteAllComponents |  |  | Delete all components. |
| importFlightLog | flFileName | params.xml | Import a trajectory file using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Import Trajectory` dialog. |
| importGroundControlPoints | gcpFileName | params.xml | Import ground control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Ground Control Points` dialog. |
| importControlPointsMeasurements | cpmFileName | params.xml | Import measurements of control points (CPs) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Control Points Measurements` dialog. |
| editControlPointSelection | "key=value" |  | Edit the settings/parameters of the selected control points based on the value in the `Selected control point(s)` panel or its key. More information can be found here . |
| listControlPoints | fileName |  | Export a list of control points to the specified file path, including the file name and extension (fileName paramater). Each control point will be listed with its index. |
| selectControlPoint | controlPointName |  | Select a control point by its name. |
| invertControlPointSelection |  |  | Invert the control point selection. If no control points are selected, all will be selected. |
| renameControlPoint | controlPointName newName |  | Rename a control point. Specify control point by its current name (controlPointName) and the new name you want to assign (newName). |
| renameSelectedControlPoint | newName |  | Rename the selected control point to the specified new name (newName). |
| deleteControlPoint |  | index | Delete a selected control point. If an optional parameter index is used, a control point with a corresponding index will be removed. Index numbers start at 0 (zero) and follow the order of control points in the 1Ds view. |
| selectMeasurementByError | errorValue | controlPointName | Select any measurement with a position error (in pixels) equal to or greater than the specified errorValue (float value). If no control point is specified (controlPointName), the selection applies to all measurements regardless of their assigned control point. |
| selectMeasurementByIndex | controlPointName index |  | Select a control point measurement by its index within the specified control point. |
| deleteControlPointMeasurement |  |  | Remove selected control point measurements (images assigned to a control point). Images have to be selected in the 1Ds view under the corresponding control point. |
| exportGroundControlPoints | gcpFileName | params.xml | Export ground control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Ground Control` dialog. |
| exportControlPointsMeasurements | cpmFileName | params.xml | Export measurements of control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Export Control Points Measurements` dialog when pressing the Shift key together with the button `Control Points` in the Export part of the Alignment tab. |
| defineDistance | PointNameA PointNameB distance | constraintName | Define a distance constraint between two control points. If constraintName is not defined, the distance name is created automatically. Alternatively, you can load distance constraints from a file (see below). |
| defineDistance | fileName | params.xml | Import a distance constraint from a file (filename including path) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the `Import Distance Definitions` dialog. The list of supported formats distancedefinitions.xml can be found in the installation folder. |
| editConstraintSelection | "key=value" |  | Edit the settings/parameters of the selected constraints based on the value in the `Selected constraint(s)` panel or its key. More information can be found here . |
| deleteConstraint |  | index | Remove the selected distance constraints. The constraints must be selected in the 1Ds view. You can use an index as an optional parameter to specify which constraint to remove. Indexes start at 0 and follow the order of constraints in the 1Ds view. |
| detectMarkers |  | params.xml | Detect markers in images using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the `Detect Markers` tool in the application. |
| setCamerasGravityDirection |  | componentID | If the image's XMP file contains gravity information (xcr:Gravity), the component will be rotated so the -z vector is in the direction of the gravity vector. This does not apply to the dense point cloud (mesh/model), only to the sparse point cloud (alignment). Will apply to the selected component or to the component defined with the optional parameters (component ID). |

## Reconstruction

These commands can be used to calculate models and perform coloring and texturing.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| resetGround |  |  | Set the ground plane back to its original orientation and position. |
| setGroundPlaneFromReconstructionRegion |  |  | Automatically center a model using a reconstruction region into the middle of the grid, adjusting both rotation and transformation. |
| setReconstructionRegionAuto |  |  | Set a reconstruction region automatically. |
| setReconstructionRegion | box.rsbox |  | Import a reconstruction region from the box.rsbox file. |
| setReconstructionRegionOnCPs | controlPoint controlPoint controlPoint controlPoint|heightValue |  | Set a reconstruction region on existing control points. Three control points define the base of the region, and the height of the region is defined by the fourth control point or by entering the height value. You can find more about this command in the examples here . |
| setReconstructionRegionByDensity |  |  | Set the reconstruction region to the part of the sparse point cloud with the highest density. |
| scaleReconstructionRegion | scaleX scaleY scaleZ | origin|center absolute|factor | Scale the reconstruction region in each axis ( `scaleX` , `scaleY` , and `scaleZ` ). Scale parameters can be treated either as absolute values ( `absolute` ) or scale factors ( `factor` ) from the center of the region ( `center` ) or its origin ( `origin` ) defined by the first control point when using `setReconstructionRegionOnCPs` command. Default values are "absolute" and "center". You can find more about this command in the examples here . |
| moveReconstructionRegion | moveX moveY moveZ |  | Move the reconstruction region along the region's axes. The values are in the coordinate system’s units. You can find more about this command in the examples here . |
| rotateReconstructionRegion | rotateX rotateY rotateZ |  | Rotate the reconstruction region around its axes. All values are in degrees. You can find more about this command in the examples here . |
| offsetReconstructionRegion | offsetX offsetY offsetZ |  | Offset the reconstruction region on its axes by the values of its dimensions. You can find more about this command in the examples here . |
| exportReconstructionRegion | box.rsbox |  | Export a reconstruction region to the box.rsbox file. |
| calculatePreviewModel |  |  | Calculate 3D mesh in the preview quality. |
| calculateNormalModel |  |  | Calculate 3D mesh in the normal quality. |
| calculateHighModel |  |  | Calculate 3D mesh in the highest quality. |
| continueModelCalculation |  |  | If the model calculation was paused or if there was a crash, it is possible to continue the calculation using this command. Find more about this command here . |

## Model Tools

When you have successfully created a model, you can use these tools to process it further by means of the command line.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| selectModel | modelName |  | Select a model with the specified name (modelName). |
| deleteSelectedModel |  |  | Delete the currently selected model. |
| duplicateSelectedModel |  |  | Duplicate the selected model (including textures). |
| renameSelectedModel | newModelName |  | Rename the currently selected model. |
| correctColors |  | layerName | Run color correction for all layers or a specified layer (layerName) in the selected component. |
| unwrap |  | params.xml | Calculate the unwrap of a model using the current settings or the settings from the params.xml (optional parameter). You can export these settings from the `Unwrap` tool in the application. |
| calculateTexture |  | params.xml | Calculate texture using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the `Color and Texture Settings` panel in the application. |
| calculateQualityTexture |  |  | Calculate texture using mesh quality values which can be calculated using the Quality Analysis tool. The values don't have to be calculated prior to using this command. |
| reprojectTexture | sourceModel resultModel | params.xml | Reproject texture from a textured model (sourceModel) to an unwrapped model (resultModel) using the current settings or the settings from the params.xml file (optional parameter). sourceModel and resultModel are model names from the application. You can export these settings from the `Texture Reprojection` tool in the application. |
| calculateVertexColors |  |  | Calculate coloring using the current settings. |
| calculatePreviewVertexColors |  |  | Calculate draft coloring using the current settings. |
| calculateQualityColors |  |  | Calculate vertex colors using mesh quality values which can be calculated using the Quality Analysis tool. The values don't have to be calculated prior to using this command. |
| simplify |  | targetTriangleCount or params.xml | Simplify a selected model using the current settings (use it without any parameters) to the selected triangle count (targetTriangleCount) or using the settings from the params.xml file. You can export these settings from the `Simplify` tool in the application. |
| smooth |  | params.xml | Smooth a selected model using the current settings or the settings from the params.xml file. You can export these settings from the `Smooth` tool in the application. |
| closeHoles |  | maxEdgesCount | Close model holes using the current settings or a specified maximal number of edges (maxEdgesCount). |
| cleanModel |  |  | Clean the selected model: remove non-manifold edges and vertices, close small holes, etc. |
| selectTrianglesInsideReconReg |  |  | Select triangles inside the reconstruction region. |
| selectTrianglesOutsideReconReg |  |  | Select triangles outside the reconstruction region. |
| selectMarginalTriangles |  |  | Select triangles that enclose the volume but are not part of the current reconstruction. |
| selectLargeTrianglesAbs | edgeSizeThreshold |  | Select triangles with edge lengths larger than the threshold (edgeSizeThreshold). |
| selectLargeTrianglesRel | edgeSizeThreshold |  | Select triangles with an edge length larger than the threshold (edgeSizeThreshold) multiplied by the average edge length. |
| selectLargestModelComponent |  |  | Select triangles that belong to the largest connected component of the model. |
| invertTrianglesSelection |  |  | Invert a selection of triangles. |
| deselectModelTriangles |  |  | Deselect all model triangles. |
| removeSelectedTriangles |  |  | Create a new model with selected triangles left out (the same behavior as achieved with the `Filter Selection` tool). |
| cutByBox | inner|outer | fillHoles | Filter out the triangles inside or outside of the reconstruction region. The required parameter defines which triangles will be filtered out, and the optional parameter fillHoles determines if the cut holes will be filled (can be set to true or false , corresponding to the Yes and No values). If the optional parameter is not chosen, the default value will be used ( Yes ). This tool creates a new model. |
| exportModel | modelName fileName | params.xml | Export a model (modelName from the project) as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export model` dialog. |
| exportSelectedModel | fileName | params.xml | Export the selected model as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export model` dialog. |
| exportModelToZip | filePath | modelFormat | Export the selected model into a compressed archive. Specify the path where the archive will be saved and the name of the exported file. Including the file extension for the archive format is optional. Additionally, you can specify the model format (e.g., .obj or .fbx) as an optional parameter. |
| importModel | fileName | params.xml | Import a model from a file. Specify the full path to the model, as well as its name and format extension. Additionally, use the settings file (exported from the import dialog) to adjust the import settings. |
| calculateOrthoProjection |  | rsorthoFile rsboxFile | Calculate an orthographic projection using the current settings. You can optionally use a projection parameters file (.rsortho) to define the calculation settings. This file can be exported together with an ortho projection, and its structure is described here . An additional optional parameter, the exported reconstruction region file (.rsbox), can be used to specify the area included in the ortho projection. |
| selectOrthoProjection | orthoName |  | Select an orthographic projection with the specified name (orthoName). |
| editOrthoProjectionSelection | "key=value" |  | Edit the settings/parameters of the selected ortho projections based on the value in the `Selected ortho photo(s)` panel or its key. More information can be found here . |
| exportOrthoProjection | orthoName fullPath params.xml |  | Export the orthographic projection (orthophoto, DSM, or DTM) using settings from the params.xml file, which can be exported from the corresponding export dialog. The command can be written in three ways, depending on which parameters are used. The params.xml file defines the export settings; orthoName is the name of the projection as it appears in the project; fullPath specifies the complete path to the exported file; folderPath defines the directory where the file will be saved; and exportName sets the name of the exported file. For fullPath and exportName , the format extension is optional—if omitted, a TIFF file is created. |
| orthoName folderPath exportName params.xml |  |  |  |
| fullPath params.xml |  |  |  |
| calculateCrossSections |  | step axis | Calculate Cross Sections either using current settings or by setting the axis (local axis of reconstruction region) and the step between each cross section. |
| exportCrossSections | fileName | params.xml | Export selected cross sections as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export Cross Sections` dialog. |
| renameCrossSections | crossSectionsName |  | Rename selected cross sections. |
| selectCrossSections | crossSectionsName |  | Select cross sections by name. |
| computeContours |  | params.xml | Compute contours for the selected Ortho using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the `Contours Tool` in the application. |
| exportContours | fileName | params.xml | Export selected contours as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export Contours` dialog. |
| renameContours | contoursName |  | Rename selected contours. |
| selectContours | contoursName |  | Select contours by name. |
| exportShapes | fileName | params.xml | Export selected shapes as a file (fileName including path and file extension .json) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export Shapes` dialog. |
| importShapesToSelectedOrtho | fileName mosaicing|measurements |  | Import shapes from a file (fileName including path and file extension) to the selected ortho. Their type will be defined based on which shape-creating tool is activated: `Measure` or `Enhance Mosaic` . |
| importShapesToOrtho | fileName orthoProjectionName mosaicing|measurements |  | Import shapes from a file (fileName including path and file extension) to ortho (orthoProjectionName). Their type will be defined based on which shape-creating tool is activated: `Measure` or `Enhance Mosaic` . |
| selectShape | shapeName |  | Select shape by name |
| addShapeToSelection | shapeName |  | Add a shape to the current selection by name. |
| exportReport | outputFileName templateFileName | true|false | Export a report into a file (outputFileName including the path and the .html extension) using a template (templateFileName including the path). The default templates are stored in the installation folder\Reports. Use the optional boolean paramater (true or false) to export a file with reports found in the specified template. |
| printReport | reportString |  | Write out report texts in the Command Prompt. This command can be used when RealityScan is run with a batch file. It does not work with delegation. Learn more about report customization here . |
| generateMaskFromMesh |  |  | Generate mask images out of the existing camera views (images) and selected model. Everything around the model as seen from the camera will be masked out. |
| exportMapsAndMask |  | folderName params.xml | Export masks generated from the camera view over the model, along with depth and normal maps for the selected images. If folderName and params.xml are not specified, the results are saved alongside the original images using the current settings. The parameters file can be exported from the export dialog for maps and masks. |
| exportLod | fileName | params.xml | Export the selected model as a linear Level-of-Detail model into a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export LoD` dialog. For export to Cesium 3D Tiles (.json) file format, use the command `export3dTiles` . |
| export3dTiles | fileName | params.xml | Export the selected model as a hierarchical Level-of-Detail model into a .json file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export LoD` dialog after selecting Cesium 3D Tiles (.json) file format. |
| exportCameraSnapshots | folderName | params.xml | Render images of a model from the camera positions using all images if none or just one is selected, or using only the selected images. Specify the path of a folder where images will be stored, and use the optional file with parameters to use settings from the export dialog. |
| exportSelectedCamerasSnapshots | folderName fileFormat | params.xml | Render images of a model from the selected camera positions. Specify the path of a folder where images will be stored and the file format of the exported images (e.g. jpg or png). Use the optional file with parameters to use settings from the export dialog. |
| renderMeshFromCustomPositionYPR | fileName params.xml|fileName width height focalLength x y z yaw pitch roll |  | Export a render of the model from a camera position with the viewpoint defined by yaw, pitch, and roll orientation. Example for a render from above of a model at 0,0,0 in local Euclidean space: RealityScan.exe -renderMeshFromCustomPositionYPR "D:/Project/render.png" 1280 720 100 0 0 150 0 0 0 |
| renderMeshFromCustomPositionLookAt | fileName params.xml|fileName width height focalLength x y z atX atY atZ | upX upY upZ | Export a render of a model from a camera position, which looks at another defined position, with an option to specify the vertical axis. Example for a side view of a model at 0,0,0 in local Euclidean space while defining the vertical axis going up the Z axis: RealityScan.exe -renderMeshFromCustomPositionLookAt "D:/Project/render.png" 1280 720 50 0 -100 0 0 0 10 0 0 1 |
| renderMeshFromCustomGridPositionYPR | fileName params.xml|fileName width height focalLength x y z yaw pitch roll |  | Export a render of the model from a custom grid position, with the viewpoint defined by yaw, pitch, and roll orientation. |
| renderMeshFromCustomGridPositionLookAt | fileName params.xml|fileName width height focalLength x y z atX atY atZ | upX upY upZ | Export a model render from a custom grid position, with the viewpoint directed toward a defined target, and the option to specify the vertical axis. |
| uploadToSketchfab | APIToken |  | Upload your model to Sketchfab. The required parameter is the Sketchfab API token and it can be found in the account settings of your Sketchfab account. |

```
RealityScan.exe -renderMeshFromCustomPositionYPR "D:/Project/render.png" 1280 720 100 0 0 150 0 0 0
```

```
RealityScan.exe -renderMeshFromCustomPositionLookAt "D:/Project/render.png" 1280 720 50 0 -100 0 0 0 10 0 0 1
```

### Classification Commands

Use these settings to create and adjust classifications.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| dtmClassify |  | params.xml | Classify vertices of the selected model into pre-defined classes using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the `Classify` tool in the application. |
| selectClassification | classificationName |  | Select a classification using its specified name (classificationName parameter). |
| renameSelectedClassification | newClassificationName |  | Rename the selected classification to the specified new name (newClassificationName). |
| transferClassification |  | params.xml | Transfer classification from the labels layer images. A file with parameters exported from the `AI Classify tool` panel can be used as an optional parameter. |
| exportClassificationSettings | XMLfilePath |  | Export settings from the `AI Classify tool` panel. For the required parameter (XMLfilePath) specify the full path for export, including the file name and format extension (.xml). |
| importClassificationSettings | XMLfilePath |  | Import settings to the `AI Classify tool` panel. For the required parameter (XMLfilePath) specify the full path for import, including the file name and format extension (.xml). |
| overrideSelectedVertices |  | className | Override the selected vertices and change their class to the currently selected class, or to the class specified by name (optional). |
| selectClass | className |  | Select a class from the selected classification using the specified class name. |
| deselectClass |  |  | Deselect all selected classes. |
| renameSelectedClass | newClassName |  | Change the name of the selected class to the specified new class name (newClassName). |
| setSelectedClassAsGroundForDTM | true OR false |  | Change the Use as ground for DTM setting in the `Selected Class` panel for the selected class. Set to true to use it as ground for DTM, or false to exclude it. |
| setSelectedClassAsGroundForExport |  |  | Change the Export class LAS setting in the `Selected Class` panel to Ground (2) . |
| setSelectedClassLasFormat | 0 - 12 |  | Change the Export class LAS setting to a value corresponding to a number from 0 to 12. |
| selectVerticesOfSelectedClass |  |  | Select model vertices based on the selected class. |
| selectClassificationFormat | classificationFormatName |  | Select the classification format based on the specified classification format name. |
| renameSelectedClassificationFormat | newClassificationFormatName |  | Rename the selected classification format to the specified new classification format name. |
| exportSelectedClassificationFormat | filePath |  | Export the selected classification format by specifying the full path, including the file name and the .cfd extension. |
| exportClassificationFormat | classificationFormatName filePath |  | Export the classification format by specifying its name and the output file path, including the file name and the .cfd extension. |
| importClassificationFormat | filePath |  | Import the classification format by specifying its full path and name, including the .cfd extension. |
| colorModelBySelectedClassification |  |  | Colorize the model according to the classes in the selected classification. |
| deleteSelectedClassification |  |  | Delete the selected classification. |

## Settings' and Error-handling Commands

You can use these commands to alter the settings and behavior of the application.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| set | "key=value" |  | Change an application setting. Learn more about how to use this command here . |
| preset | "key=value" |  | Change an application setting during the setup phase. Ideal for changes that require a reset of the application. Learn more about how to use this command here . |
| reset | ui | cfg | cfgui | all |  | Reset the user interface, settings, or both. It is also possible to make RealityScan like a clean install. This command works only when used in a batch file, and it won't work with delegation commands. What is going to be reset is determined by the chosen parameter: ui - reset user interface, cfg - reset application settings, cfgui - reset both interface and settings, all - make it like a clean install. |
| silent | crashReportPath |  | Suppress warning dialogs and uploading of the crash reports. The application will store reports in the specified location instead of showing the upload wizard. This command has to be used at the startup. |
| writeProgress | fileName | timeout | Write any new progress change into a specified file (fileName including the path). Optional timeout parameter will also output during a defined period of time (timeout in seconds). More about the file structure can be found in the section Error-handling Commands . |
| printProgress |  | timeout | Print progress change into the Windows Command Prompt for any new change. Optional timeout parameter will also output during a defined period of time (timeout in seconds). |
| tag |  |  | Writes out a tag into the Windows Command Prompt. It respects the order of the used commands and will be run after the process ran before it finishes. |
| stdConsole |  |  | Enables console redirection to the application standard output. When used, you will see the application console content mirrored also in the standard Windows console. This also enables further redirections for CLI purposes. |
| disableOnlineCommunication |  |  | Disable any online communication. |
| importGlobalSettings | settings.rcconfig |  | Import application global settings from the settings.rcconfig file. |
| exportGlobalSettings | settings.rcconfig |  | Export application global settings to the settings.rcconfig file. |
| setProjectCoordinateSystem | authority:id |  | Set a project coordinate system defined by an authority, and its ID (can be found in a specific database, e.g. epsg.xml and local.xml). An example can be found here . |
| setOutputCoordinateSystem | authority:id |  | Set an output coordinate system defined by an authority, and its ID (can be found in a specific database, e.g. epsg.xml and local.xml). An example can be found here . |

## More about the CLI

See the tutorial on the command-line interface

## List of Shortcuts - RealityScan Help

# List of Shortcuts

## Applies Everywhere

| ctrl+n | New Project | Start a new project. |
| --- | --- | --- |
| ctrl+o | Open Project | Open an existing project. |
| ctrl+s | Save Project | Save the currently opened project. |
| ctrl+shift+s | Save As Project | Save the currently opened project into a different location. |
| ctrl+shift+enter | Add Folder | Add all images in the selected folder. |
| ctrl+enter | Add Images | Add one, more or all images. |
| ctrl+shift+p | Open Panels | Open all panels related to the currently selected items. |
| F1 | Help | Open this help. |
| F3 | Control Points | Add ground control points and matching image points. |
| F4 | Constraints | Define distance between control points. |
| F5 | Start | Create a complete 3D model. |
| shift+F5 | Abort Processing | Abort the currently running process. |
| F6 | Align Images | Calculate poses for all inputs. |
| ctrl+F6 | Draft Alignment | Quickly estimate poses for all inputs. |
| ctrl+F7 | Preview Reconstruction | Create a low detail 3D model very quickly. |
| F7 | Reconstruct Model | Create a 3D model in normal detail. |
| F8 | Color Model | Calculate a color for every triangle vertex. |
| F9 | Texture Model | Calculate a model texture. |
| F11 | Ortho Projection | Open the ortho projection tool/panel. |
| ctrl+a | Select all | Select all cameras or other objects. |
| ctrl+i | Invert selection | Deselect the current selection and at the same time select all objects which had not been selected. |
| ctrl+d | Clear selection | Deselect all objects. |
| ctrl+z, alt+backspace | Undo | Undo the last action. |
| ctrl+y | Redo | Revert the effects of the undo action. |
| ctrl+shift+~ | Screen recording | Start/stop a recording of a workflow in the application window. |
| ctrl+shift+F1/F2/F3/F4/F5 | Remember Image Selection | Remember up to 5 created image selections. |
| ctrl+shift+1/2/3/4/5 | Set Image Selection | Reuse previously remembered image selections. |

## Applies To a Selected View

| ctrl+1 | Blue | Select the blue cursor in the active view. |
| --- | --- | --- |
| ctrl+2 | Green | Select the green cursor in the active view. |
| ctrl+3 | Magenta | Select the magenta cursor in the active view. |
| ctrl+4 | Coral | Select the coral cursor in the active view. |
| hold space | Camera | Hold the space bar during any action to switch to a camera tool to move/pan/rotate/etc. the view camera. |

## Applies When Selecting Items

| shift | Add | Add all items starting from the last previously selected item to the current selection. |
| --- | --- | --- |
| ctrl | Toggle | Add a new item into an existing selection or remove the item if it is already in the selection. |
| ctrl+shift | Remove | Remove one or more items from the current selection. |
| hold 1 | Make Blue | Hold 1 when clicking on an item to assign it to the blue cursor. Not every item can be assigned. |
| hold 2 | Make Green | Hold 2 when clicking on an item to assign it to the green cursor. Not every item can be assigned. |
| hold 3 | Make Magenta | Hold 3 when clicking on an item to assign it to the magenta cursor. Not every item can be assigned. |
| hold 4 | Make Coral | Hold 4 when clicking on an item to assign it to the coral cursor. Not every item can be assigned. |

## Applies To Image View 2D

| → | Next | Show next image. |
| --- | --- | --- |
| ← | Previous | Show previous image. |
| Page Up | Previous by 20 | Show the previous twentieth image. |
| Page Down | Next by 20 | Show the next twentieth image. |
| SHIFT+ → | Next under component | Show the next image in the selected component. |
| SHIFT+ ← | Previous under component | Show the previous image in the selected component. |
| ↓ | Next under control point | Show the next image of the selected control point. |
| ↑ | Previous under control point | Show the previous image of the selected control point. |
| F | Frame Selection | Frame a view to the selected control points. |
| R | Reset View | Reset a view to its default state. |
| ctrl+C | Copy Filename to Clipboard | Copy the filename of a selected image to clipboard. |
| tab | Switch Layer | Switch between image layers, e.g. image and intensity images, textures or depth maps. |
| D/E | Disable / Enable | Disable/enable the selected image. |
| CTRL+Insert | Add to selection | Add the displayed images into the current selection. |
| CTRL+Delete | Remove from selection | Remove the displayed images from the current selection. |
| CTRL+7 | View Epipolar Geometry | Activates the ray-of-sight tool. |
| CTRL+6 | View Features | Show image features / tie points. |

## Applies To Ortho View 2D

| [ | Decrease | Deacrease the size of a brush when creating a shape on the ortho projection. |
| --- | --- | --- |
| ] | Increase | Increase the size of a brush when creating a shape on the ortho projection. |

## Applies in Scene View 3D

| F | Frame selection | Place the pivot in the middle of selection. If nothing is selected, zoom the camera to the pivot. |
| --- | --- | --- |
| A | Frame all | Move the camera to see all. |
| C | Select camera by a lasso | Select cameras by the camera lasso tool. |
| shift+C | Select camera by a rectangle | Select cameras by the camera rect(angle) tool. |
| X | Find points from cameras | After camera selection, mark all sparse points seen by selected cameras. |
| D | Select points by a lasso | Applicable just to aligned models which have not been reconstructed yet. |
| shift+D | Select points by a rectangle | Applicable just to aligned models which have not been reconstructed yet. |
| S | Find cameras | Marks all cameras seeing selected sparse points. |
| E | Expand point selection | Expand your selection of 3D points visible by a set of cameras with all points seen by these cameras. |
| B | Create a clipping box | Restrict rendering to a box. |
| shift+B | Clear the clipping box | Remove the created clipping box. |
| ctrl+B | Edit the clipping box | Show and adjust the created clipping box. |
| ctrl+shift+U | Show the reconstruction region | Show and adjust the reconstruction region. |
| R | View reset | Reset the 3D view to its original position and rotation. |
| I | Component inspector | Activate the Inspect tool. |
| P | Center the pivot | Set the pivot in the middle of a selection. |
| shift+P | Clear the pivot | Completely remove the pivot from the scene. |
| escape | Exit an action. | Deactivate the current context widget (selection etc.) |
| ctrl+L | Lock camera pose(s) for alignment | Lock selected camera poses in a component. |
| ctrl+R | Disable an image | Disable/enable an image selected in the 3D view for further processing. |
| ctrl+E | Enable displaying cameras | Enable/disable displaying positions of cameras. |
| ctrl+H | Hide selected cameras | Hide selected cameras. |
| shift+H | Show selected cameras | Show hidden selected cameras. |
| ctrl+shift+H | Unhide all cameras | Unhide all previously hidden cameras. |
| enter | Set the camera view | Set the view camera to the pose of a camera selected by the cursor |
| 0 (numpad) | Set the 3D view to a perspective view | Show a perspective of a model. |
| 1 (numpad) | Set the 3D view to a parallel view | Show a parallel projection of a model. |
| 2 (numpad) | Set the 3D view to the top view | Show the top of a model. |
| 3 (numpad) | Set the 3D view to the bottom view | Show the bottom of a model. |
| 4, 5, 6, 7 (numpad) | Set the 3D view to a side view | Show side(s) of a model. |

## Applies To Console

| ctrl+C | Copy to Clipboard | Copy the content of the console to clipboard. |
| --- | --- | --- |

## General Gestures

| right-mouse button | Back/Cancel | Return to the previous step or cancel the tool. |
| --- | --- | --- |
| hold shift+run the application | Reset settings | Reset the application settings to default. |

## Model Tools - RealityScan Help

# Model Tools

When you have successfully created a model, you can use these tools to colorize your model, create textures, and do some post-processing on your models.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| selectModel | modelName |  | Select a model with the specified name (modelName). |
| deleteSelectedModel |  |  | Delete the currently selected model. |
| duplicateSelectedModel |  |  | Duplicate the selected model (including textures). |
| renameSelectedModel | newModelName |  | Rename the currently selected model. |
| correctColors |  | layerName | Run color correction for all layers or a specified layer (layerName) in the selected component. |
| unwrap |  | params.xml | Calculate unwrap of a model using the current settings or the settings from the params.xml (optional parameter). You can export these settings using the `Unwrap` tool in the application. |
| calculateTexture |  |  | Calculate texture using the current settings. |
| calculateQualityTexture |  |  | Calculate texture using mesh quality values which can be calculated using the Quality Analysis tool. The values don't have to be calculated prior to using this command. |
| reprojectTexture | sourceModel resultModel | params.xml | Reproject texture from a textured model (sourceModel) to an unwrapped model (resultModel) using the current settings or the settings from the params.xml file (optional parameter). sourceModel and resultModel are model names from the application. You can export these settings from the `Texture Reprojection` tool in the application. |
| calculateVertexColors |  |  | Calculate coloring using the current settings. |
| calculatePreviewVertexColors |  |  | Calculate draft coloring using the current settings. |
| calculateQualityColors |  |  | Calculate vertex colors using mesh quality values which can be calculated using the Quality Analysis tool. The values don't have to be calculated prior to using this command. |
| simplify |  | targetTriangleCount or params.xml | Simplify a selected model using the current settings (use it without any parameters) to the selected triangle count (targetTriangleCount) or using the settings from the params.xml file. You can export these settings using the `Simplify Tool` in the application. |
| smooth |  | params.xml | Smooth a selected model using the current settings or the settings from the params.xml file. You can export these settings from the `Smoothing Tool` in the application. |
| closeHoles |  | maxEdgesCount | Close model holes using the current settings or using a specified maximal number of edges (maxEdgesCount). |
| cleanModel |  |  | Clean the selected model: remove non-manifold edges and vertices, close small holes, etc. |
| selectTrianglesInsideReconReg |  |  | Select triangles inside the reconstruction region. |
| selectTrianglesOutsideReconReg |  |  | Select triangles outside the reconstruction region. |
| selectMarginalTriangles |  |  | Select triangles that are enclosing the volume but are not part of the current reconstruction. |
| selectLargeTrianglesAbs | edgeSizeThreshold |  | Select triangles with an edge length larger than the threshold (edgeSizeThreshold). |
| selectLargeTrianglesRel | edgeSizeThreshold |  | Select triangles with an edge length larger than the threshold (edgeSizeThreshold) multiplied by average edge length. |
| selectLargestModelComponent |  |  | Select triangles that belong to the largest connected component of the model. |
| invertTrianglesSelection |  |  | Invert a selection of triangles. |
| deselectModelTriangles |  |  | Deselect all model triangles. |
| removeSelectedTriangles |  |  | Create a new model with selected triangles left out (the same behavior as with the `Filter Selection` tool). |
| cutByBox | inner|outer | fillHoles | Filter out the triangles inside or outside of the reconstruction region. The required parameter defines which triangles will be filtered out, and the optional parameter fillHoles determines if the cut holes will be filled (can be set to true or false which correspond to the Yes and No values). If the optional parameter is not chosen, the default value will be used ( Yes ). This tool creates a new model.. |
| exportModel | modelName fileName | params.xml | Export a model (modelName from the project) as a file (fileName including the path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export this file from the `Export model` dialog. |
| exportSelectedModel | fileName | params.xml | Export the selected model as a file (fileName including the path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export model` dialog. |
| exportModelToZip | filePath | modelFormat | Export the selected model into a compressed archive. Specify the path where the archive will be saved and the name of the exported file. Including the file extension for the archive format is optional. Additionally, you can specify the model format (e.g., .obj or .fbx) as an optional parameter. |
| importModel | fileName |  | Import a model from a file (fileName including the path and file extension). |
| calculateOrthoProjection |  | rsorthoFile rsboxFile | Calculate an orthographic projection using the current settings. You can optionally use a projection parameters file (.rsortho) to define the calculation settings. This file can be exported together with an ortho projection, and its structure is described here . An additional optional parameter, the exported reconstruction region file (.rsbox), can be used to specify the area included in the ortho projection. |
| selectOrthoProjection | orthoName |  | Select an orthographic projection with the specified name (orthoName). |
| editOrthoProjectionSelection | "key=value" |  | Edit the settings/parameters of the selected ortho projections based on the value in the `Selected ortho photo(s)` panel or its key. More information can be found here . |
| exportOrthoProjection | orthoName fullPath params.xml |  | Export the orthographic projection (orthophoto, DSM, or DTM) using settings from the params.xml file, which can be exported from the corresponding export dialog. The command can be written in three ways, depending on which parameters are used. The params.xml file defines the export settings; orthoName is the name of the projection as it appears in the project; fullPath specifies the complete path to the exported file; folderPath defines the directory where the file will be saved; and exportName sets the name of the exported file. For fullPath and exportName , the format extension is optional—if omitted, a TIFF file is created. |
| orthoName folderPath exportName params.xml |  |  |  |
| fullPath params.xml |  |  |  |
| calculateCrossSections |  | step axis | Calculate Cross Sections either using current settings or by setting the axis (local axis of reconstruction region) and the step between each cross section. |
| exportCrossSections | fileName | params.xml | Export selected cross sections as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export Cross Sections` dialog. |
| renameCrossSections | crossSectionsName |  | Rename selected cross sections. |
| selectCrossSections | crossSectionsName |  | Select cross sections by name. |
| computeContours |  | params.xml | Compute contours for the selected Ortho using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the `Contours Tool` in the application. |
| exportContours | fileName | params.xml | Export selected contours as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export Contours` dialog. |
| renameContours | contoursName |  | Rename selected contours. |
| selectContours | contoursName |  | Select contours by name. |
| exportShapes | fileName | params.xml | Export selected shapes as a file (fileName including path and file extension .json) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the `Export Shapes` dialog. |
| importShapesToSelectedOrtho | fileName mosaicing|measurements |  | Import shapes from a file (fileName including path and file extension) to the selected ortho. Their type will be defined based on which shape-creating tool is activated: `Measure` or `Enhance Mosaic` . |
| importShapesToOrtho | fileName orthoProjectionName mosaicing|measurements |  | Import shapes from a file (fileName including path and file extension) to ortho (orthoProjectionName). Their type will be defined based on which shape-creating tool is activated: `Measure` or `Enhance Mosaic` . |
| selectShape | shapeName |  | Select shape by name |
| addShapeToSelection | shapeName |  | Add a shape to the current selection by name. |
| exportReport | outputFileName templateFileName | true|false | Export a report into a file (outputFileName including the path and the .html extension) using a template (templateFileName including the path). The default templates are stored in the installation folder\Reports. Use the optional boolean paramater (true or false) to export a file with reports found in the specified template. |
| printReport | reportString |  | Write out report texts in the Command Prompt. This command can be used when RealityScan is run with a batch file. It does not work with delegation. Learn more about report customization here . |
| generateMaskFromMesh |  |  | Generate mask images out of the existing camera views (images) and selected model. Everything around the model as seen from the camera will be masked out. |
| exportDepthAndMask |  | folderName params.xml | Export depth maps and/or masks for selected images. If folderName and params.xml are not specified, results are stored along with the original images using the current settings. You can export the settings file from the application in the `Export Depth Maps and Masks` dialog. |
| exportLod | fileName | params.xml | Export the selected model as a linear Level-of-Detail model into a file (fileName including the path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export this file from the `Export LoD` dialog. For export to Cesium 3D Tiles (.json) file format use command `export3dTiles` . |
| export3dTiles | fileName | params.xml | Export the selected model as a hierarchical Level-of-Detail model into a .json file (fileName including the path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export this file from the `Export LoD` dialog after selecting Cesium 3D Tiles (.json) file format. |
| exportCameraSnapshots | folderName | params.xml | Render images of a model from the camera positions using all images if none or just one is selected, or using only the selected images. Specify the path of a folder where images will be stored, and use the optional file with parameters to use settings from the export dialog. |
| renderMeshFromCustomPositionYPR | fileName params.xml|width height focalLength x y z yaw pitch roll |  | Export a render of the model from a camera position using yaw, pitch, and roll orientation. Example for a view from above a model at 0,0,0 in local Euclidean space: RealityScan.exe -renderMeshFromCustomPositionYPR "C:/images/image.png" 1280 720 100 0 0 150 0 0 0 |
| renderMeshFromCustomPositionLookAt | fileName params.xml|width height focalLength x y z atX atY atZ | upX upY upZ | Export a render of the model from a camera position, which looks at another XYZ coordinate, with an option to add another XYZ coordinate to determine which direction the vertical axis ascends to. Example for a side view of a model at 0,0,0 in local Euclidean space, using the up option: RealityScan.exe -renderMeshFromCustomPositionLookAt "C:/images/image.png" 1280 720 50 0 -100 0 0 0 10 0 0 1 |
| renderMeshFromCustomGridPositionYPR | fileName params.xml|fileName width height focalLength x y z yaw pitch roll |  | Export a render of the model from a custom grid position, with the viewpoint defined by yaw, pitch, and roll orientation. |
| renderMeshFromCustomGridPositionLookAt | fileName params.xml|fileName width height focalLength x y z atX atY atZ | upX upY upZ | Export a model render from a custom grid position, with the viewpoint directed toward a defined target, and the option to specify the vertical axis. |
| uploadToSketchfab | APIToken |  | Upload your model to Sketchfab. The required parameter is the Sketchfab API token and it can be found in the account settings of your Sketchfab account. |

```
RealityScan.exe -renderMeshFromCustomPositionYPR "C:/images/image.png" 1280 720 100 0 0 150 0 0 0
```

```
RealityScan.exe -renderMeshFromCustomPositionLookAt "C:/images/image.png" 1280 720 50 0 -100 0 0 0 10 0 0 1
```

### Classification Commands

Use these settings to create and adjust classifications.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| dtmClassify |  | params.xml | Classify vertices of the selected model into pre-defined classes using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the `AI Classify tool` in the application. |
| selectClassification | classificationName |  | Select a classification using its specified name (classificationName parameter). |
| renameSelectedClassification | newClassificationName |  | Rename the selected classification to the specified new name (newClassificationName). |
| transferClassification |  | params.xml | Transfer classification from the labels layer images. A file with parameters exported from the `AI Classify tool` panel can be used as an optional parameter. |
| exportClassificationSettings | XMLfilePath |  | Export settings from the `AI Classify tool` panel. For the required parameter (XMLfilePath) specify the full path for export, including the file name and format extension (.xml). |
| importClassificationSettings | XMLfilePath |  | Import settings to the `AI Classify tool` panel. For the required parameter (XMLfilePath) specify the full path for import, including the file name and format extension (.xml). |
| overrideSelectedVertices |  | className | Override the selected vertices and change their class to the currently selected class, or to the class specified by name (optional). |
| selectClass | className |  | Select a class from the selected classification using the specified class name. |
| deselectClass |  |  | Deselect all selected classes. |
| renameSelectedClass | newClassName |  | Change the name of the selected class to the specified new class name (newClassName). |
| setSelectedClassAsGroundForDTM | true OR false |  | Change the Use as ground for DTM setting in the `Selected Class` panel for the selected class. Set to true to use it as ground for DTM, or false to exclude it. |
| setSelectedClassAsGroundForExport |  |  | Change the Export class LAS setting in the `Selected Class` panel to Ground (2) . |
| setSelectedClassLasFormat | 0 - 12 |  | Change the Export class LAS setting to a value corresponding to a number from 0 to 12. |
| selectVerticesOfSelectedClass |  |  | Select model vertices based on the selected class. |
| selectClassificationFormat | classificationFormatName |  | Select the classification format based on the specified classification format name. |
| renameSelectedClassificationFormat | newClassificationFormatName |  | Rename the selected classification format to the specified new classification format name. |
| exportSelectedClassificationFormat | filePath |  | Export the selected classification format by specifying the full path, including the file name and the .cfd extension. |
| exportClassificationFormat | classificationFormatName filePath |  | Export the classification format by specifying its name and the output file path, including the file name and the .cfd extension. |
| importClassificationFormat | filePath |  | Import the classification format by specifying its full path and name, including the .cfd extension. |
| colorModelBySelectedClassification |  |  | Colorize the model according to the classes in the selected classification. |
| deleteSelectedClassification |  |  | Delete the selected classification. |

## Model Tools Examples

These are some examples of the model tools, import and export usage.

### Simplify, smooth, texture and export a model

This piece of code will calculate a model in a normal detail, simplify it to 100k triangles, smooth, texture and then export the result.

```
RealityScan.exe .... -calculateNormalModel -simplify 100000 -smooth -calculateTexture -exportModel "Model 3" %MyPath%\Object.obj %MyPath%\objParams.xml -save %Mypath%\NewProject.rsproj -quit
```

NOTE: You can obtain the parameters file from the `Export Model` dialog. The project now contains three models: the first one is a normal model, the second one is simplified, and the third one is simplified, smoothed and textured at the same time. By default, the last created model is selected for the next processing. You can now select a model by its name using the command `-selectModel` and then export this model using the command `-exportSelectedModel` . This piece of code will calculate a model in a normal detail, simplify it to 100k triangles, smooth, texture but export the second model:

```
RealityScan.exe .... -calculateNormalModel -simplify 100000 -smooth -calculateTexture -selectModel “Model 2” -exportSelectedModel %MyPath%\Object.obj %MyPath%\objParams.xml -save %Mypath%\NewProject.rsproj -quit
```

### Import models to a component

Now suppose you have a folder called Models and you want to import all of them to an aligned project (saving it as a new project).

```
RealityScan.exe -load %MyPath%\AlignedProject.rsproj -save %MyPath%\NewProject.rsproj -quit
for %%s in (%MyPath%\Models\*.obj) do (
RealityScan.exe -load %MyPath%\NewProject.rsproj -selectMaximalComponent -importModel "%%s" -save %MyPath%\NewProject.rsproj -quit
)
```

The For cycle will iterate through all files with the .obj extension and add them to the project.

### Calculate an orthographic projection

When you create and export an orthographic projection by the command, you have to attach the files which contain all necessary parameters.

```
RealityScan.exe .... -calculateNormalModel -calculateOrthoProjection %MyPath%\params.rcortho -exportOrthoProjection %MyPath%\ortho.tiff exportOrthoParams.xml -save %MyPath%\MyProject.rsproj -quit
```

You can obtain the `params.rcortho` file when you export an orthographic projection from the GUI and set Export projection parameters file to True . To learn more about export and import parameter files, visit this page .

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Commands Outside Command Prompt

Using CLI with an .rscmd file

## Delegation of Commands

On delegating commands into an opened instance of RealityScan

## Alignment Commands

Commands for alignment and component handling

## Reconstruction Commands

Model calculation via the command line

## Continue

Application settings and behaviour

## Error Handling Commands

Commands for handling potential errors

### See also:

- Learn about a model export click here
- Learn about a model import click here
- See the orthographic projection tutorial click here
- See all CLI commands in one page click here

## Parametric Files of Orthographic Projections - RealityScan Help

# Parametric Files of Orthographic Projections

When creating orthographic projections using the command `-calculateOrthoProjection` , a parametric file with the `.rcortho` extension is used to define parameters of a rendered orthographic projection. Let us take a closer look at its content.

## *.rcortho for Calculation of Orthographic Projections

When using the command `-calculateOrthoProjection` , the application loads the parameters from a specified params.rcortho file. The file can be generated in a suitable format by creating an orthographic projection manually, exporting it into any format, and setting exportProjectionParametersFile to True during the export. An example of the .rcortho file content for calculation of an orthographic projection:

```
<OrthoProjection width="10011" height="8076" name="Ortho projection 1" modelName="Model 1"
    colorType="texturing" boxSideConerIndex="13" bEmpty="0" backFaceColorType="1" backFaceColor="2130706687">
    <Header magic="5787472" version="1"/>
</OrthoProjection>
<ReconstructionRegion globalCoordinateSystem="NONE" globalCoordinateSystemName="NONE" isGeoreferenced="0"
    isLatLon="0" yawPitchRoll="0 -0 -0">
    <widthHeightDepth>29.8926887512207 29.9313926696777 24.1154346466064</widthHeightDepth>
    <Header magic="5395016" version="2"/>
    <CentreEuclid>
        <centre>-0.098011314868927 0.0846212208271027 12.4095182418823</centre>
    </CentreEuclid>
    <Residual R="1 0 0 0 1 0 0 0 1" t="0 0 0" s="1"/>
</ReconstructionRegion>
<DTMParams classificationLayerId="-1">
  <ClassificationParams modelType="nature" postprocessType="soft_edges" sensitivity="0.5"/>
  <Header magic="1480868688" version="1"/>
</DTMParams>
```

We can generate the file in a suitable format by creating an orthographic projection manually, exporting it into any format and setting exportProjectionParametersFile to True when exporting. The first part of the file indicates the parameters needed to create a projection.

### Parameters:

- width/height - a width/height of a projection
- name - a name of a projection
- modelName - a name of the model which we want to calculate the projection for
- boxSideConerIndex - a number from 0 to 23; it defines which side of the reconstruction region will be the projected plane, and which will be the upper-left corner of the final projection; this number is not defined definitely because it depends on the rotation of the reconstruction region; the best procedure, how to obtain this number, is to create one orthographic projection manually and export it with the projection parameters' file; when making further projections, the use of the number from the boxSideConerIndex will suffice
- colorType - alternatives: texturing, coloring
- bEmpty - alternatives: 0 - an orthographic projection will be calculated; an equivalent to pressing the Render button in the Ortho Projection Tool settings. 1 - an orthographic projection will not be calculated; an equivalent to pressing the Add to batch button in the Ortho Projection Tool settings. We recommend you to leave this parameter set to 0
- backFaceColorType - alternatives: 0 - None - the inner parts of a model projection are not colored differently from the outer ones. 1 - FixedColor - the inner parts of a model projection are colored differently from the outer ones; the color is defined in the backFaceColor.
- backFaceColor - the color of the inner parts of a model

The second part describes parameters of a reconstruction region whose one side represents a projected plane. This part can be obtained by setting the reconstruction region manually and then exporting it.

The third part (between "DTMParams" tags) contains the parameters needed to create a digital terrain model (DTM).

### Parameters:

- classificationLayerId - ID of a classification layer from which the DTM is created. "-1" means that a new classification is calculated during rendering. In such case, the user needs to define also ClassificationParams (see below).
- modelType - type of a classification based on the nature of the scene. Alternatives: "industrial_complex", "mixed", "city", "nature", "meadows", "countryside", "mountains".
- postprocessType - type of the postprocessor that will execute a process after classification to clean a model. Alternatives: "none", "soft_edges", "hard_edges".
- sensitivity - value between 0 and 1. If set to 0, everything will be classified as "Artificial object", if set to 1, everything will be classified as "Ground".

To learn more about the point cloud classification, visit this page .

## Learn How to Use Them

See a tutorial for the command line interface

## Process IDs - RealityScan Help

# Process IDs

The `writeProgress` CLI command can be used with several parameters, one of which is `algId` . The `algId` parameter returns the process ID of the process running at the time `writeProgress` was executed.

| Process ID | Process name |
| --- | --- |
| 0 | SUBPROCESS_NO_ACTION |
| 5 | EXPORT_VIDEO |
| 6 | EXPORT_MODEL |
| 7 | MODEL_TEXTURE |
| 8 | MODEL_COLORIZE |
| 9 | MARK_TRIANGLES |
| 10 | FILTER_SELECTED_TRIANGLES |
| 11 | SIMPLIFY |
| 12 | SMOOTH |
| 13 | EXPORT_DEPTH_MAPS |
| 14 | EXPORT_MASK |
| 15 | EXPORT_RENDER |
| 16 | VISIBILITY |
| 17 | IMPORT_MODEL |
| 18 | UPLOAD_CRASH_REPORT |
| 19 | DOWNLOAD_NEW_VERSION |
| 20 | IMPORT_IMAGE_SELECTION |
| 21 | RECOMPUTE_MODEL_RENDER_DATA |
| 22 | EVALUATING |
| 23 | CHECK_MODEL_INTEGRITY |
| 24 | FILTERING_CORRUPTED_DATA |
| 25 | CLEAN_MODEL |
| 26 | CLOSE_HOLES |
| 27 | UNDERCUT_MODEL_PARTS |
| 28 | CHECK_MODEL_TOPOLOGY |
| 29 | DUPLICATE_MODEL |
| 30 | DETECT_MARKERS |
| 31 | GENERATE_MARKERS |
| 32 | MERGING_PART_PAIRS |
| 33 | MERGING_SIBLING_PARTS_IN_DECOMPOSITION |
| 34 | CREATING_EXTERNAL_TRIANGLES_FOR_MODEL_PARTS |
| 35 | EXPORTING_TO_PTX |
| 36 | EXPORT_DEPTH_AND_MASK_IMAGES |
| 40 | IMPORT_IMAGES_FROM_URL |
| 41 | DETECT_SHARPREGION |
| 42 | AI_CLASSIFY |
| 43 | EXPORT_ST_MAPS |
| 44 | SIMPLIFY_GROUP |
| 45 | REMOVE_TEXTURES |
| 47 | TRANSFER_IMAGE_LABELS |
| 48 | OVERRIDE_CLASSIFICATION |
| 50 | MODEL_BASED_COLORNORMALIZATION |
| 51 | AI_OVERRIDE_CLASSIFICATION |
| 4144 | GROUPINGPARSE |
| 8208 | DEPTH_MAPS |
| 8240 | MESHING |
| 8242 | CLUSTERING |
| 20532 | PROJECT_LOAD |
| 20533 | PROJECT_SAVE |
| 20534 | PROJECT_AUTOSAVE |
| 20560 | CALCULATE_MODEL_PREVIEW |
| 20561 | CALCULATE_MODEL_NORMAL |
| 20562 | CALCULATE_MODEL_HIGH |
| 20563 | CORRECT_COLORS |
| 20564 | CREATE_ORTHO_PROJECTION |
| 20565 | RENDER_ORTHOS_IN_BATCH |
| 20566 | SHARE_RESULTS |
| 20567 | EXPORT_REPORT |
| 20568 | EXPORTING_RIGGING_XMP_FILES |
| 20569 | EXPORT_CP_MEASUREMENTS |
| 20576 | EXPORT_REGISTRATION |
| 20577 | EXPORT_NORMALIZED_IMAGES |
| 20578 | EXPORT_ORTHO_PHOTO |
| 20579 | EXPORT_ORTHO_PHOTO_SINGLE_SELECTION |
| 20583 | EXPORT_ORTHO_PHOTO |
| 20584 | EXPORT_XMP |
| 20585 | EXPORT_POINT_CLOUD |
| 20586 | EXPORT_DEPTH_AND_MASK |
| 20587 | EXPORT_PPI_LICENSE_FILE |
| 20588 | IMPORT_PPI_LICENSE_FILE |
| 20589 | ACQUIRE_PPI_LICENSE |
| 20590 | COLOR_NORMALIZATION_AUTOMATIC |
| 20591 | DETECT_IMAGE_LABELS |
| 20592 | IMPORT_LASER_SCAN |
| 20594 | IMPORT_COMPONENT |
| 20597 | IMPORT_COMPONENT_STRUCTURE |
| 20598 | IMPORT_FLIGHT_LOG |
| 20599 | IMPORT_GCP |
| 20600 | IMPORT_CP_MEASUREMENTS |
| 20601 | CONTINUE_MODEL_CALCULATION |
| 20640 | CHANGE_COORDINATE_SYSTEM |
| 20735 | INTERACT_START_BUTTON |
| 20736 | COMPUTING_MODEL_PARAMS |
| 20737 | UNWRAP_MODEL |
| 20738 | UNWRAP_MODEL |
| 20739 | UNWRAP_MODEL |
| 20740 | UNWRAP_MODEL |
| 20741 | UNWRAP_MODEL |
| 20742 | FILL_TEXTURES |
| 20744 | EXPORT_MODEL_CUTS |
| 21000 | EXPORT_GLOBAL_CONFIG |
| 21001 | IMPORT_GLOBAL_CONFIG |
| 21024 | EXPAND_SELECTION |
| 21025 | EXPAND_CONNECTED_COMPONENTS_SELECTION |
| 21026 | CALCULATE_COMPONENTS_METADA |
| 21028 | SELECT_MARGINAL_TRIANGLES |
| 21029 | SELECT_TRIANGLES_BY_EDGE_SIZE |
| 21030 | SELECT_MAX_CONNECTED_COMPONENTS |
| 21040 | REPROJECT_TEXTURE |
| 21056 | CLOUD_REPORT |
| 21776 | RESET_GROUND_PLANE |
| 21777 | SET_GROUND_PLANE |
| 21778 | SET_GROUND_PLANE_BY_RECONSTRUCTION_REGION |
| 21779 | IMPORT_RECONSTRUCTION_REGION |
| 21780 | SET_RECONSTRUCTION_REGION_ON_RECONSTRUCTION |
| 21781 | SET_RECONSTRUCTION_REGION_GRID |
| 21782 | SET_RECONSTRUCTION_REGION_BEST_NORTH_SOUTH |
| 21783 | SET_RECONSTRUCTION_REGION_BEST_FIT |
| 21784 | SET_RECONSTRUCTION_REGION_CLIPPING_BOX |
| 21785 | SET_RECONSTRUCTION_REGION_AUTO |
| 21786 | SET_RECONSTRUCTION_REGION_CP |
| 21787 | SELECT_TIE_POINTS_RECT |
| 21788 | SELECT_TIE_POINTS_LASSO |
| 21789 | SELECT_CAMERAS_RECT |
| 21790 | SELECT_CAMERAS_LASSO |
| 21793 | SELECT_VERTICES_BOX |
| 21794 | SELECT_VERTICES_LASSO |
| 21795 | SELECT_VERTICES_RECT |
| 21796 | SELECT_ALL_VERTICES |
| 21797 | INVERT_VERTEX_SELECTION |
| 21798 | SELECT_ALL_CAMERAS |
| 21799 | INVERT_CAMERAS_SELECTION |
| 21800 | EXPORT_RECONSTRUCTION_REGION |
| 21801 | EXPORT_IMAGE_LIST |
| 21802 | EXPORT_SKETCHFAB |
| 21807 | INSPECT |
| 21808 | IMPORT_LEICA_BLK3D |
| 21809 | DEFINE_DISTANCE |
| 21810 | COLLECT_METADATA_AT_STARTUP |
| 21811 | ATOMIC_TRIANGLE_SELECTION |
| 21812 | EXPORT_UNDISTORTED_IMAGES |
| 21813 | EXPORT_CESIUM |
| 21814 | EXPORT_GCP |
| 21815 | CLEAR_RECONSTRUCTION_REGION |
| 21816 | INTERACT_DESELECT_CAMERAS_TIEPOINTS_SELECTION |
| 21845 | CLI_PARSE_PARAMS |
| 21857 | CLI_SELECT_COMPONENT |
| 21859 | CLI_RENAME_SELECTED_COMPONENT |
| 21861 | CLI_CLEAR_CACHE |
| 21863 | CLI_DELETE_SELECTED_COMPONENT |
| 21876 | CLI_EXPORT_MODEL |
| 21877 | CLI_SET_SELECTED_INPUTS_PROPERTY |
| 21881 | CLI_IMPORT_HDR_IMAGES |
| 21882 | CLI_RCCMD_EXEC |
| 21884 | CLI_SELECT_IMAGE |
| 22529 | ADD_CONTROL_POINT |
| 22530 | REMOVE_CONTROL_POINT |
| 24576 | UPLOAD_TO_SKETCHFAB |
| 24580 | CALCULATE_FILE_SIZE |
| 24657 | HELP_SEARCH |
| 24660 | UNZIP_SAMPLE_DATASET |
| 24661 | ADDING_INPUTS |
| 24848 | IMPORT_VIDEO |
| 24867 | MAPWIZARD_JOB |
| 24868 | MAPWIZARD_OPEN |
| 24869 | MAPWIZARD_START_PROCESSING |
| 28672 | EXPORT_LOD |
| 28677 | IMPORT_HDR_IMAGES |
| 32768 | HELP_URL |
| 32769 | HELP_OPEN |
| 41061 | EXPORT_REGISTRATION_FILE |
| 41062 | EXPORT_REGISTRATION_COMPONENT |
| 41063 | EXPORT_REGISTRATION_PREPROCESS |
| 41064 | EXPORT_REGISTRATION_FINALIZE |
| 65536 | IMPORT_IMAGES |
| 65537 | ALIGN_NORMAL |
| 65538 | ALIGN_DRAFT |
| 65539 | SFM_FEATURES_DETECTION |
| 65542 | UPDATE_CONSTRAINTS |
| 77824 | SFM_MATCHING |
| 77840 | SFM_ALIGNMENT_MAIN |

## RealityScan Node - RealityScan Help

# RealityScan Node

Node is a system service acting as a bridge between a custom application and RealityScan. It supports REST-like API for managing RealityScan sessions, uploading and downloading session files, and running CLI commands.

RealityScan Node can be started either from RealityScan, or directly from the RealityScan installation folder. To run it from RealityScan, go to the `Assistants` part in the WORKFLOW tab and use the Real-time Assistance tool. This should automatically start a RealityScan Node (you can find it in the Windows System Tray). To start it from the RealityScan installation folder, just run the RSNode.exe file or call it from the command prompt. Here is an example:

```
"C:\Program Files\Epic Games\RealityScan\RSNode.exe" -hostAddress 192.168.0.74 -port 7878 -landingPage "\/static/MyApp.html"
```

Currently, there is a single method of authentication, using a Bearer HTTP Authorization header with a GUID token. This token is sent as a GET parameter when opening the landing page. This token is required by all API calls (except the static HTTP server). If you are connecting to a node directly, note that the connection is not secure, and therefore relies on a private and secured Local Area Network.

## API Methods - Node

The API methods affecting the node

## API Methods - Project

The API methods affecting the project

## Reconstruction Commands - RealityScan Help

# Reconstruction Commands

The commands below can be used to set and modify the reconstruction region, and run the reconstruction.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| resetGround |  |  | Set the ground plane back to its original orientation and position. |
| setGroundPlaneFromReconstructionRegion |  |  | Automatically center a model using a reconstruction region into the middle of the grid, adjusting both rotation and transformation. |
| setReconstructionRegionAuto |  |  | Set a reconstruction region automatically. |
| setReconstructionRegion | box.rsbox |  | Import a reconstruction region from the box.rsbox file. |
| setReconstructionRegionOnCPs | controlPoint controlPoint controlPoint controlPoint|heightValue |  | Set a reconstruction region on existing control points. Three control points define the base of the region and the height of the region is defined by the frouth control point or by entering the height value. See the example below. |
| setReconstructionRegionByDensity |  |  | Set the reconstruction region to the part of the sparse point cloud with the highest density. |
| scaleReconstructionRegion | scaleX scaleY scaleZ | origin|center absolute|factor | Scale the reconstruction region in each axis ( `scaleX` , `scaleY` , and `scaleZ` ). Scale parameters can be treated either as absolute values ( `absolute` ) or scale factors ( `factor` ) from the center of the region ( `center` ) or its origin ( `origin` ) defined by the first control point when using `setReconstructionRegionOnCPs` command. Default values are "absolute" and "center". See the examples below. |
| moveReconstructionRegion | moveX moveY moveZ |  | Move the reconstruction region along the region's axes. The values are in the coordinate system’s units. See the examples below. |
| rotateReconstructionRegion | rotateX rotateY rotateZ |  | Rotate reconstruction region around its axes. All values are in degrees. See the examples below. |
| offsetReconstructionRegion | offsetX offsetY offsetZ |  | Offset the reconstruction region on its axes by the values of its dimensions. |
| exportReconstructionRegion | box.rsbox |  | Export a reconstruction region to the box.rsbox file. |
| calculatePreviewModel |  |  | Calculate 3D mesh in the preview quality. |
| calculateNormalModel |  |  | Calculate 3D mesh in the normal quality. |
| calculateHighModel |  |  | Calculate 3D mesh in the highest quality. |
| continueModelCalculation |  |  | If model calculation was paused, or if there was a crash, it is possible to continue calculation using this command. |

## Reconstruction Examples

The examples which follow show how to run model calculations from the command line.

We will begin with a simple script that shows the first two reconstruction-region commands in function.

```
RealityScan.exe -addFolder %MyPath%\Images\ -align -setReconstructionRegionAuto -exportReconstructionRegion %MyPath%\myBox.rsbox -quit
```

After basic alignment, it will set a reconstruction region automatically and export it to a file. If you want to import a custom reconstruction region, you can first export it from the application by clicking on MESH & COLOR / `Export` / `Reconstruction Region` and then alter the values and parameters inside the .rsbox file. The script below will open an aligned project, select the largest component, import a reconstruction region and calculate models in a preview, normal and high detail.

```
RealityScan.exe -load %MyPath%\AlignedProject.rsproj -selectMaximalComponent -setReconstructionRegion %MyPath%\myBox.rsbox -calculatePreviewModel -calculateNormalModel -calculateHighModel -save %MyPath%\NewProject.rsproj -quit
```

```
RealityScan.exe -load %MyPath%\AlignedProject.rsproj^
-selectMaximalComponent -setReconstructionRegion %MyPath%\myBox.rsbox^
-calculatePreviewModel^
-calculateNormalModel^
-calculateHighModel^
-save %MyPath%\NewProject.rsproj -quit
```

The multiline script should do the same as the one-line script, but it is more legible and separated into logical parts.

## Set Reconstruction Region on Control Points

You can use command `-setReconstructionRegionOnCPs` to set a reconstruction region with control points. It is necessary to have at least three control points set to use this command. These three points will define the base plane of the reconstruction region (first two control points define the width of the region and the third one defines the length). The height of the reconstruction region can be set either by fourth control point or entering the height value (in the coordinate system’s units). This value can be negative.

Example:

```
RealityScan.exe -delegateTo * -setReconRegionOnCPs CP1 CP2 CP3 50
```

This example will create a reconstruction region on control points CP1, CP2, and CP3 with the height 50 in the coordinate system’s units. The order of the control points in the command defines the reconstruction region’s axes. The axes always define a right-handed system which has the origin in the first defined control point and X axis starts in the origin and goes through the second control point.

## Scale the reconstruction region

Reconstruction region can be scaled on its axes based on the absolute values or multiplying factors using the command `-scaleReconstructionRegion` . It is also possible to choose from where it will be scaled: origin and center. The default settings for this command are set to scale the reconstruction region from the center of the region and based on the absolute values.

Example:

```
RealityScan.exe -delegateTo * -scaleReconstructionRegion 1.1 1.1 1.2 center factor
```

This example will scale reconstruction region from the center based on the defined factors. Axes X and Y will be multiplied by the factor 1.1 and Z axis will be multiplied by the factor 1.2.

## Move the reconstruction region

To move the reconstruction region the command `-moveReconstructionRegion` can be used.

Example:

```
RealityScan.exe -delegateTo * -moveReconstructionRegion 10 10 10
```

When used, this example will move reconstruction region by 10 units (defined by the coordinate system) on all axes.

## Rotate the reconstruction region

Rotation of the reconstruction region is executed with the command `-rotateReconstructionRegion` . Defined parameters needed are all in degrees.

Example:

```
RealityScan.exe -delegateTo * -rotateReconstructionRegion 45 45 45
```

This example will rotate reconstruction region by 45 degrees on all axes, which also will be rotated.

## Offset the reconstruction region

Offset the reconstruction region with the command `-offsetReconstructionRegion` by the length of its sides. The parameters are relative values that serve as multiplicators.

Example:

```
RealityScan.exe -delegateTo * -offsetReconstructionRegion 1 2 0.5
```

This example will offset the reconstruction region on the X axis by one length of its depth, on the Y axis by two lengths of its width, and on the Z axis by half of the height length. These values are applied to the reconstruction region's XYZ axes.

## Precomputation of Depth Maps

To speed up the model computation process using more computers, you can precompute depth maps with the PrecomputeDepthmaps setting:

```
RealityScan.exe -set "PrecomputeDepthmaps=true"
```

- If the value true is set, then the preview/normal/high command will just precompute depth maps, which are stored in the application cache, and no model will be created.
- If the default value false is set, then the preview/normal/high command will perform as usual, i.e. depth maps will be computed and a model will also be created. However, if depth maps have already been precomputed and are stored in the cache, then the depth-map computation part will be skipped and only the model computation process will be performed.

This approach is useful when you wish to maximize GPU/CPU utilization and if you have many assets to compute, which are of almost the same size, for example numerous full-body scans. You can build two machines. One with multiple powerful GPUs but normal CPU, and the second with normal GPU but powerful CPU. Then you will need RealityScan CLI on both machines. Having this setup, you may keep running these two pipelines in parallel:

- GPU-PC, Dataset(i+1) : Align \ Turn On Precompute Depth Maps \ Model Computation \ Save \ Move the project and cache to CPU-PC: RealityScan.exe -addFolder "C:\MyFolder_on_GPU-PC\Images\"^ -align^ -set "PrecomputeDepthmaps=true"^ -calculateNormalModel^ -save "C:\MyFolder_on_GPU-PC\Project.rsproj" -quit Move `Project.rsproj` from `C:\MyFolder_on_GPU-PC` to `C:\MyFolder_on_CPU-PC` and cache to `C:\MyFolder_on_CPU-PC\Cache` .
- CPU-PC, Dataset(i) : Load \ Import Cache \ Turn Off Precompute Depth Maps \ Model Computation \ Export: RealityScan.exe -load "C:\MyFolder_on_CPU-PC\Project.rsproj"^ -importCache "C:\MyFolder_on_CPU-PC\Cache\"^ -set "PrecomputeDepthmaps=false"^ -calculateNormalModel^ -exportModel "Model 1" "C:\MyFolder_on_CPU-PC\Model.obj" "C:\MyFolder_on_CPU-PC\params.xml"^ -save "C:\MyFolder_on_CPU-PC\Project_Model.rsproj" -quit

```
RealityScan.exe -addFolder "C:\MyFolder_on_GPU-PC\Images\"^
-align^
-set "PrecomputeDepthmaps=true"^
-calculateNormalModel^
-save "C:\MyFolder_on_GPU-PC\Project.rsproj" -quit
```

Move `Project.rsproj` from `C:\MyFolder_on_GPU-PC` to `C:\MyFolder_on_CPU-PC` and cache to `C:\MyFolder_on_CPU-PC\Cache` .

```
RealityScan.exe -load "C:\MyFolder_on_CPU-PC\Project.rsproj"^
-importCache "C:\MyFolder_on_CPU-PC\Cache\"^
-set "PrecomputeDepthmaps=false"^
-calculateNormalModel^
-exportModel "Model 1" "C:\MyFolder_on_CPU-PC\Model.obj" "C:\MyFolder_on_CPU-PC\params.xml"^
-save "C:\MyFolder_on_CPU-PC\Project_Model.rsproj" -quit
```

## Continue Model Calculation

When there is an unfinished model due to a crash or model calculation was paused, it is possible to continue its calculation using the command `-continueModelCalculation` . To be able to use this command after a crash, auto save mode must be activated beforehand. Command scans for unfinished models in your project, and once it encounters one it continues its calculation. Models are searched in the order from the last component to the first one, and within them from the last model to the first model.

Example (when paused):

```
RealityScan.exe -load project.rsproj -continueModelCalculation
```

When pausing a calculation, it is possible to choose not to pause it temporarily. This will leave model in a partially computed state. This example will continue the calculation of that model if the loaded project was saved before closing.

Example (after a crash):

```
RealityScan.exe -load project.rsproj recoverAutosave -continueModelCalculation
```

As mentioned above, auto save mode must be enabled to be able to use this command. When it is enabled, it is possible to use an optional parameter with the command `-load` , which is `recoverAutosave` . This parameter will enable RealityScan to scan for the unfinished model.

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Commands Outside Command Prompt

Using CLI with an .rscmd file

## Delegation of Commands

On delegating commands into an opened instance of RealityScan

## Alignment Commands

Commands for alignment and component handling

## Continue

Further model processing via the command line

## Settings' Commands

Application settings and behaviour

## Error Handling Commands

Commands for handling potential errors

### See also:

- Learn about model and reconstruction settings click here
- See all CLI commands in one page click here

## Settings' Commands - RealityScan Help

# Settings' Commands

With these commands you can alter the application settings and behaviour directly from the command line.

| Command Name | Required Parameter | Optional Parameter | Description |
| --- | --- | --- | --- |
| set | "key=value" |  | Set an application state variable. |
| preset | "key=value" |  | Change an application setting during the setup phase. Ideal for changes that require a reset of the application. Learn more about how to use this command here . |
| reset | ui | cfg | cfgui | all |  | Reset the user interface, settings, or both. It is also possible to make RealityScan like a clean install. This command works only when used in a batch file, and it won't work with delegation commands. What is going to be reset is determined by the chosen parameter: ui - reset user interface, cfg - reset application settings, cfgui - reset both interface and settings, all - make it like a clean install. |
| silent | crashReportPath |  | Set a location for storing crash reports. The application will store the reports here instead of showing the upload wizard. |
| writeProgress | fileName | timeout | Write any new progress change into a specified file (fileName including the path). Optional timeout parameter will also output during a defined period of time (timeout in seconds). More about the file structure can be found in the section Error-handling Commands . |
| printProgress |  | timeout | Print progress change into the Windows Command Prompt for any new change. Optional timeout parameter will also output during a defined period of time (timeout in seconds). |
| tag |  |  | Writes out a tag into the Windows Command Prompt. It respects the order of the used commands and will be run after the process ran before it finishes. |
| stdConsole |  |  | Enables console redirection to the application standard output. When used, you will see the application console content mirrored also in the standard Windows console. This also enables further redirections for CLI purposes. |
| disableOnlineCommunication |  |  | Disable any online communication. |
| importGlobalSettings | settings.rsconfig |  | Import application global settings from the settings.rsconfig file. |
| exportGlobalSettings | settings.rsconfig |  | Export application global settings to the settings.rsconfig file. |
| setProjectCoordinateSystem | authority:id |  | Set a project coordinate system defined by an authority, and its ID (can be found in a specific database, e.g. epsg.xml and local.xml). |
| setOutputCoordinateSystem | authority:id |  | Set an output coordinate system defined by an authority, and its ID (can be found in a specific database, e.g. epsg.xml and local.xml). |

## Examples of the Settings

```
RealityScan.exe -set "sfmDistortionModel=Brown3" ^
-set "unwrapMaximalTexCount=1" ^
-set "unwrapStyle=1"
```

With the following commands, you can set the local Euclidean coordinate system as a project coordinate system, and the GPS (WGS84) with EPSG code 4326 as an output coordinate system:

```
RealityScan.exe -setProjectCoordinateSystem Local:1 ^
-setOutputCoordinateSystem epsg:4326
```

## Project and Image Commands

Manage the current project, the application itself and add images via CLI

## Commands Outside Command Prompt

Using CLI with an .rscmd file

## Delegation of Commands

On delegating commands into an opened instance of RealityScan

## Alignment Commands

Commands for alignment and component handling

## Reconstruction Commands

Model calculation via the command line

## Model Tools' Commands

Further model processing via the command line

## Continue

Commands for handling potential errors

### See also:

- See all CLI commands in one page click here
- See the application settings keys and values click here

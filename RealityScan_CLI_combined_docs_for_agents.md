# RealityScan / RealityCapture CLI Guide for Agents

## What This File Covers

This file is the one-stop CLI reference for an agent that needs to operate RealityScan / RealityCapture, not just look up isolated command names.

It is split into two layers:

- An agent-first operations guide that explains how to think about state, sequencing, selections, and parameter artifacts.
- A complete CLI command reference generated from the official HTML help pages in this folder.

The command appendix is anchored to the master `List of All CLI Commands` page, so every command listed there is included here.

## Core Operating Model

RealityScan CLI is a sequential control surface for a stateful application.

- Commands execute in the order you pass them.
- Each command changes application state.
- Later commands depend on project state created by earlier commands.
- Correct syntax is necessary, but correct state is what usually determines success.

The CLI behaves like driving the GUI in order: load or create a project, add data, change selections, run a computation, then export from the resulting state.

## State Dependencies

Most broken runs come from the right command being executed against the wrong state. Track these explicitly:

- Current project: whether you are in a new scene or a loaded `.rsproj` / `.rcproj` file.
- Loaded inputs: images, scans, image lists, imported components, or models.
- Selected images: many input-level commands operate on the current selection.
- Selected component: downstream reconstruction depends on the component you choose.
- Selected model: many model tools and exports operate on the current model.
- Reconstruction region: model calculation is bounded by the current or imported region.
- Settings and presets: runtime behavior changes based on `-set` and other configuration commands.
- External parameter files: some workflows depend on GUI-exported files such as `params.xml`, `.rsbox`, or `.rcortho`.

Before writing a sequence, answer: what project is loaded, what is selected, what file-based parameters are required, and which command changes that state next?

## Task-to-Command Map

Use this as the fast index for common agent tasks.

- Create a project from images: `-newScene`, `-add` or `-addFolder`, then `-align`, then `-save`.
- Reuse an alignment: align once, export with `-exportXMP` or `-exportRegistration`, then import or reapply to a new project.
- Build a mesh: select the intended component, set or import the reconstruction region, then run `-calculatePreviewModel`, `-calculateNormalModel`, or `-calculateHighModel`.
- Export a final model: ensure the correct model is selected, then use `-exportModel`, `-exportSelectedModel`, or `-exportModelToZip`.
- Generate ortho deliverables: prepare a `.rcortho` file, run `-calculateOrthoProjection`, then export ortho outputs.
- Run unattended: use `-headless` or `-hideUI`, plus progress and error-handling commands.
- Coordinate multiple running instances: assign names, then use delegation commands only after the single-instance flow is already correct.

## Standard Workflow Lifecycle

A reliable default workflow for most CLI jobs is:

1. Start a new scene or load an existing project.
2. Add inputs.
3. Apply required settings and import configuration artifacts.
4. Align and inspect resulting components.
5. Select the correct component explicitly.
6. Set, import, or adjust the reconstruction region.
7. Compute one or more models.
8. Post-process the model if needed.
9. Export deliverables.
10. Save, monitor progress, and quit cleanly.

## Inputs and Project Setup

The project and input layer is where the run is defined.

- Use `-newScene`, `-load`, `-save`, and `-quit` to control project lifecycle.
- Use `-add`, `-addFolder`, and `.imagelist` files for standard image ingestion.
- Use `-importVideo` to extract frames from video at a fixed interval.
- Use `-importLeicaBlk3D`, `-importLaserScan`, and `-importLaserScanFolder` for supported scan sources.
- Use `-importHDRimages` when you need the HDR import path and optional import settings.
- Use `-addImageWithCalibration` when an image must be paired with its XMP calibration file.
- Use image-selection commands before changing per-input flags like alignment, meshing, texturing, or masking.
- Use cache commands intentionally: importing cache can accelerate repeat work, while clearing cache requires the project to be saved first.

An agent should treat input setup as part of the workflow definition, not just a prelude. Wrong inputs, wrong image selection, or wrong import parameters propagate into every later stage.

## Alignment and Component Control

Alignment is the first major branch point because it determines whether you have usable components.

- `-detectFeatures` and `-align` are the normal starting points.
- `-draft` is the faster, lower-fidelity alignment path.
- `-mergeComponents` combines existing components without adding new images.
- Alignment creates one or more components; most downstream work should explicitly choose one.
- Use `-selectMaximalComponent`, `-selectComponent`, or `-selectComponentWithLeastReprojectionError` before reconstruction.
- Use component export and import commands to split work across projects or machines.
- Use control-point, marker, GCP, and constraint commands when the workflow depends on measured references.
- Use registration export commands when alignment needs to be reused outside the current project.

A large share of CLI failures are really component-selection mistakes. If the wrong component is active, model calculation and export may still run, but on the wrong data.

## Reconstruction Region and Model Calculation

Reconstruction depends on both component choice and region definition.

- Use `-setReconstructionRegionAuto` for a quick region based on the selected component.
- Use `-setReconstructionRegion`, `-setReconstructionRegionOnCPs`, `-moveReconstructionRegion`, `-rotateReconstructionRegion`, `-scaleReconstructionRegion`, and `-offsetReconstructionRegion` to define or refine the region.
- Use `-exportReconstructionRegion` and `-importReconstructionRegion` to reuse a region through an `.rsbox` file.
- Use `-calculatePreviewModel`, `-calculateNormalModel`, or `-calculateHighModel` based on fidelity and time budget.
- Use `-continueModelCalculation` for interrupted or long-running model work.
- Use depth-map and preview-color commands where a partial or staged workflow is better than full reconstruction immediately.

The dependency chain is: selected component -> reconstruction region -> model calculation -> selected model.

## Model Processing and Deliverables

Once a model exists, the CLI expands into post-processing and export.

- Use simplification, smoothing, filtering, hole-closing, and triangle-selection commands to shape the mesh.
- Use coloring, texturing, and UV-related commands to produce usable surface outputs.
- Use model import and duplication commands when comparing alternative branches.
- Use model export commands for standard geometry outputs, zipped outputs, LoD, and 3D tiles.
- Use ortho, contours, cross sections, shapes, masks, depth maps, and camera snapshot commands when the deliverable is not just a mesh.
- Many exports accept `params.xml` files generated by the GUI export dialogs; these files are often part of the real workflow, not optional noise.

By default, the most recently created model tends to become the active one. If multiple models exist, explicitly reselect before further processing or export.

## Settings, Presets, and External Parameter Files

There are three different kinds of configuration inputs an agent must keep straight.

- Runtime settings: commands such as `-set` that change application keys and behavior.
- Operation-specific commands: commands that directly change selection, grouping, or import/export behavior.
- External parameter files: GUI-exported configuration files used as command parameters.

Key practical file types:

- `params.xml`: commonly used for import and export dialogs, including model, registration, point cloud, mask, and scan workflows.
- `.rsbox`: reconstruction region definition for reusable meshing bounds.
- `.rcortho`: orthographic projection parameters for `-calculateOrthoProjection`.

For ortho workflows in particular, `.rcortho` is not optional if you want reproducible CLI behavior. The file is typically generated by exporting an orthographic projection from the GUI with projection-parameter export enabled.

## Reliability, Headless Runs, and Monitoring

For unattended automation, explicit observability matters.

- Use `-headless` to suppress the UI from startup.
- Use `-hideUI` and `-showUI` when you need to toggle visibility without true headless startup.
- Use `-silent` to redirect crash-report output to a target folder.
- Use `-writeProgress` or `-printProgress` to surface process state during long runs.
- Use `-set "appQuitOnError=true"` when the caller should fail fast on errors.
- Use autosave-handling options on `-load` when an autosaved project may exist.
- Use `-quit` or `-restart` intentionally at the end of scripted sequences.

A useful rule: do not treat headless execution as complete until progress output, error policy, and exit behavior are all explicitly defined.

## Delegation and Multi-Instance Control

Delegation is useful only after the single-instance workflow is stable.

- RealityScan supports multiple open instances.
- Use `-setInstanceName` to give each instance a stable name.
- Use `-delegateTo` to route commands into a specific running instance or the first available instance.
- Use `-waitCompleted` to block until delegated work finishes.
- Use `-getStatus` to poll progress for running work.
- Use `-pauseInstance`, `-unpauseInstance`, and `-abortInstance` to control active work.
- Use `-execRSCMDIndirect` when a command file should execute inside an already opened instance.

Keep delegation late in the automation design. If the underlying sequence is wrong, multi-instance control just makes it harder to diagnose.

## Command Files (.rscmd / .rccmd)

Command files are a transport and batching mechanism, not the core conceptual model.

- Use `.rscmd` or `.rccmd` when the same sequence should be packaged and reused.
- Execute them with `-execRSCMD` or `-execRSCMDIndirect`.
- Arguments can be passed in and referenced as variables such as `$(arg0)` through `$(arg9)`.
- The files can hold one command per line or multiple commands in sequence.
- Working-directory context still matters because relative paths resolve from the process start context, not from your intention.
- Understanding state and command ordering is still more important than understanding the wrapper file.

## Non-CLI Source Pages (Brief Notes)

A few HTML pages in the source folder are not central to the CLI command surface.

- `List of Shortcuts` is GUI shortcut reference material and is intentionally not copied into this CLI guide.
- `RealityScan Node` documents a Node/API surface. It is acknowledged as related automation material, but it is outside the core CLI command reference here.
- The Node page includes RSNode startup flags `-hostAddress`, `-port`, and `-landingPage`. They are service-launch arguments for `RSNode.exe`, not standard RealityScan project commands.
- A few HTML examples use legacy or shorthand command names that are not part of the master command table: `-exportComponent`, `-minComponentSize`, and `-setReconRegionOnCPs`. Treat them as example-era aliases for `-exportSelectedComponentDir` / `-exportSelectedComponentFile`, `-setMinComponentSize`, and `-setReconstructionRegionOnCPs`.
- `Process IDs` is folded into monitoring guidance instead of being expanded into a standalone chapter.

## Complete CLI Command Reference

The following appendix is generated from the official `List of All CLI Commands` HTML page and normalized into markdown. Every command from that master list is included.
### Project and Images

- `-headless`
  Description: Hides user interface. Hides the user interface from startup.
- `-hideUI`
  Description: Hide user interface. Unlike headless, this command doesn't need to be run at startup and does not suppress actions that require user interaction.
- `-showUI`
  Description: Shows hidden user interface. Unlike headless, this command doesn't need to be run at startup.
- `-newScene`
  Description: Create a new empty scene.
- `-load`
  Required: MyProject.rsproj
  Optional: recoverAutosave|deleteAutosave
  Description: Load an existing project from the MyProject.rsproj file. Use optional parameters to define the action if there is an autosaved file present for this project. Using recoverAutosave will open the autosaved project, while deleteAutosave will delete the autosaved project and load the original one. More information about the Autosave feature can be found here. | To set the preference globally for all projects, use set command with appAutoSaveCliHandling key. Find more information in the section CLI Settings Keys & Values.
- `-save`
  Optional: MyProject.rsproj
  Description: Save the current project to its original location or save as MyProject.rsproj.
- `-start`
  Description: Run the processes configured for the Start button. Adjust these settings in the Start button settings which can be accesed from the Start button dropdown menu.
- `-unlockPPIProject`
  Required: myProject.rcproj
  Description: Save and unlock your PPI projects to make them compatible for use with the latest RealityScan versions. Use the whole path with the project's name and extension where you want to store the unlocked project.
- `-add`
  Required: imageName
  Description: Import one or more images from a specified file path or from an image list. The image list is a text file with the .imagelist extension, containing full paths to the images, each on a separate line.
- `-addFolder`
  Required: folderName
  Description: Add all images to the specified folder. To include subdirectories, use the command set with a key appIncSubdirs as follows: -set "appIncSubdirs=true". Find more information in the section CLI Settings Keys & Values.
- `-importVideo`
  Required: videoFileName | extractedVideoFramesLocation | jumpsLength
  Description: Import frames extracted from a video (videoFileName including the path). The frames are extracted into a folder (extractedVideoFramesLocation) using an interval between frames defined by the jumpsLength (in seconds).
- `-importLeicaBlk3D`
  Required: fileName
  Description: Import image sequence with .cmi extension (fileName including path) captured by Leica BLK3D.
- `-importLaserScan`
  Required: laserscanName
  Optional: params.xml
  Description: Add a LiDAR scan or a LiDAR scan list using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the LiDAR Scan Import dialog.
- `-importLaserScanFolder`
  Required: folderName
  Optional: params.xml
  Description: Add all LiDAR scans in the specified folder using settings from the params.xml file. You can export these settings from the application in the LiDAR Scan Import dialog.
- `-importHDRimages`
  Required: fileName|folderName|imageList
  Optional: params.xml
  Description: Import HDR image (imageName), list of images (imageList) or all images from a folder (folderName) using the current settings or the settings from params.xml. You can export these settings from the application in the 16-bit/HDR Images Import dialog.
- `-addImageWithCalibration`
  Required: fileName | xmpFileName
  Description: Import an image as well as the corresponding XMP file. Use whole paths to the files.
- `-importImageSelection`
  Required: fileName
  Description: Select some scene images and/or LiDAR scans listed in a file (filename including the path).
- `-selectImage`
  Required: imagePath|regexp
  Optional: set | union | sub | intersect | toggle
  Description: Select a specified image (imagePath) or images defined by regular expression (regexp). See below examples. One of the optional parameters may be used to more select images with more flexibility.
- `-selectAllImages`
  Description: Select all images in the project.
- `-deselectAllImages`
  Description: Deselect all images in the project.
- `-invertImageSelection`
  Description: Invert the current image selection.
- `-removeCalibrationGroups`
  Description: Clear all inputs from their calibration groups.
- `-generateAIMasks`
  Description: Use AI Masking to generate masks by isolating the object of interest in your images.
- `-exportMasks`
  Required: folderPath params.xml
  Description: Export the mask images currently used in the project. Specify the output folder and, optionally, the full path to the XML file with export parameters (this file can be exported from the Export Mask Images dialog).
- `-setImageLayer`
  Required: index pathImage layerType
  Description: Set the layer from the image defined with the pathImage parameter (path to the layer image) to the image defined with the index parameter. The index corresponds to the image order in the 1Ds view, starting at 0 (zero). The layerType parameter defines which layer to set onto the chosen image (e.g., mask, texture).
- `-setImagesLayer`
  Required: pathImage layerType
  Description: Set the layer from the image defined with the pathImage parameter (path to the layer image) to the selection of images. The layerType parameter defines which layer to set onto the selected images (e.g., mask, texture).
- `-removeImageLayer`
  Required: layerType
  Description: Remove the layers corresponding to the layerType parameter (e.g. mask, texture) from the selected images.
- `-importCache`
  Required: folderName
  Description: Import resource cache data from a specified folder.
- `-clearCache`
  Description: Clear the application cache. You must save the project before clearing the application cache.
- `-execRSCMD`
  Required: Commands.rscmd
  Description: Execute commands from an .rscmd (or .rccmd file), optionally using report parameters. The file format and syntax are described in the Commands Outside Command Prompt section. You can define up to nine arguments, acting as variables, by listing them after the path to the .rscmd file. Inside the file, reference these variables using $(arg1)-$(arg9).
- `-quit`
  Description: Quit the application.
- `-setFeatureSource`
  Required: 0|1|2
  Description: Define a feature source mode for the selected images: | 0 - Merge using overlaps, | 1 - Use component features, | 2 - Use all image features.
- `-enableAlignment`
  Required: true|false
  Description: Enable/disable selected images in the registration process.
- `-enableMeshing`
  Required: true|false
  Description: Enable/disable selected images in the model computation/meshing.
- `-enableTexturingAndColoring`
  Required: true|false
  Description: Enable/disable selected images during the coloring and texture calculation.
- `-setWeightInTexturing`
  Required: <0,1>
  Description: Set weight for selected images during the coloring and texture calculation.
- `-enableColorNormalizationReference`
  Required: true|false
  Description: Set selected images as color references in the color normalization process. The colors for these images will not be changed.
- `-enableColorNormalization`
  Required: true|false
  Description: Enable or disable selected images in the color normalization process.
- `-setDownscaleForDepthMaps`
  Required: integer
  Description: Set a downscale factor for depth-map computation for the selected images.
- `-enableInComponent`
  Required: true|false
  Description: Enable selected images in meshing and continue. Applicable only for the registered images.
- `-setCalibrationGroupByExif`
  Description: Set the calibration group of all inputs based on their Exif.
- `-setConstantCalibrationGroups`
  Description: Group all selected inputs into a single calibration group.
- `-lockPoseForContinue`
  Required: true|false
  Description: Set relative camera pose unchanged for the selected images during the next registration. Applicable only for the registered images.
- `-setPriorCalibrationGroup`
  Required: number
  Description: Set a prior calibration group for the selected images: | -1 - do not group, | another number - group into the same calibration group.
- `-setPriorLensGroup`
  Required: number
  Description: Set a prior lens group for the selected images. Using -1 means do not group; any other number means to group the selected images into the same distortion group.
- `-editInputSelection`
  Required: "key=value"
  Description: Edit the settings of the selected inputs based on the value in the Selected inputs panel or its key. More information can be found here.

### Delegate commands

- `-setInstanceName`
  Required: instanceName
  Description: Assign a name to a RealityScan instance.
- `-delegateTo`
  Required: instanceName|*
  Description: Delegate a command or a sequence of commands to a specific instance of RealityScan (instanceName) or to the first available instance (using the * symbol as a parameter).
- `-waitCompleted`
  Required: instanceName|*
  Description: Pause execution of other commands until the current process is finished in a specified instance of RealityScan (instanceName) or in the first available instance (using the * as a parameter).
- `-getStatus`
  Required: instanceName|*
  Description: Return the progress status of a running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter).
- `-pauseInstance`
  Required: instanceName|*
  Description: Pause a currently running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter).
- `-unpauseInstance`
  Required: instanceName|*
  Description: Unpause a currently paused process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter).
- `-abortInstance`
  Required: instanceName|*
  Description: Abort a currently running process in a specified instance of RealityScan (instanceName) or in the first available instance (using the * symbol as a parameter). If processing is done using the CLI commands, all processes after the running process will also be aborted.
- `-execRSCMDIndirect`
  Required: instanceName|* commands.rscmd
  Description: Execute commands listed in the .rscmd (or .rccmd) file in the specified instance, optionally using report parameters. The file format and syntax are described in the Commands Outside Command Prompt section. You can define up to nine arguments, acting as variables, by listing them after the path to the .rscmd file. Inside the file, reference these variables using $(arg1)-$(arg9).

### Alignment

- `-align`
  Description: Align images using the current settings.
- `-draft`
  Description: Align images in the draft mode using the current settings.
- `-update`
  Description: Update all components and models by a rigid transformation to fit the actual constraints and control points.
- `-detectFeatures`
  Description: Run feature detection according to the alignment settings. Detected features will be saved in the application cache.
- `-mergeComponents`
  Description: Merge already created components. When using this command, no new images are added to the existing components.
- `-exportXMP`
  Optional: params.xml
  Description: Export camera metadata of components created in the last alignment in the XMP format using the current settings or the settings from params.xml file (optional parameter). You can export this file from the XMP metadata export dialog. The components must fulfill the condition defined by the command setMinComponentSize. | NOTE: XMP files are stored in the same folder as the respective images.
- `-exportXMPForSelectedComponent`
  Description: Export camera metadata of a selected component in XMP format using the current settings. An example is available in the paragraph "Metadata (XMP) Export Settings" below. | NOTE: XMP files are stored in the same folder as the respective images.
- `-importComponent`
  Required: component.rsalign
  Description: Import a component from the component.rsalign file.
- `-importBundler`
  Required: filePath
  Optional: params.xml
  Description: Import a Bundler project. The command requires the path to the Bundler file and, optionally, a configuration file that defines the scene transformation settings saved from the import dialog. The configuration file can be used to adjust the coordinate system or apply custom transformations during import.
- `-importColmap`
  Required: filePath
  Optional: params.xml
  Description: Import a COLMAP project. The command requires the path to the COLMAP file (any of the three text files) and, optionally, a configuration file that defines the scene transformation settings saved from the import dialog. The configuration file can be used to adjust the coordinate system or apply custom transformations during import.
- `-exportLatestComponents`
  Required: folderName
  Description: Export components created in the last alignment as RealityScan Alignment Components (.rsalign) into a specified folder. The components must fulfill the condition defined by the command setMinComponentSize.
- `-setMinComponentSize`
  Required: size
  Description: Specify the minimal component size for export when using the exportLatestComponents and exportXMP commands. The default value is 5.
- `-exportSelectedComponentDir`
  Required: folderName
  Description: Export the selected component into a folder (folderName including the path) as a RealityScan Alignment Component (.rsalign).
- `-exportSelectedComponentFile`
  Required: fileName
  Description: Export the selected component into a RealityScan Alignment Component file.
- `-exportRegistration`
  Required: fileName
  Optional: params.xml
  Description: Export registration to a specified file using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Export Registration dialog.
- `-exportUndistortedImages`
  Required: folderName
  Optional: params.xml
  Description: Export undistorted images into a specified folder using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Export Registration dialog.
- `-exportSTMap`
  Optional: folderName params.xml
  Description: Export ST maps for the selected images. If folderName and params.xml are not specified, results are stored along with the original images using the current settings. You can export the settings file from the application in the Export Registration dialog.
- `-exportSparsePointCloud`
  Required: fileName
  Optional: params.xml
  Description: Export 3D tie points (a sparse point cloud) into a specified file (fileName including the path and format extension) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application within the Export Point Cloud dialog.
- `-selectComponent`
  Required: componentName
  Description: Select a component with the specified name (componentName) for further processing.
- `-selectMaximalComponent`
  Description: Select the largest component for further processing.
- `-selectComponentWithLeastReprojectionError`
  Description: Select the component with the smallest reprojection error, based on the calculated Mean error [pixels], for further processing.
- `-renameSelectedComponent`
  Required: newComponentName
  Description: Rename the currently selected component.
- `-deleteSelectedComponent`
  Description: Delete the currently selected component.
- `-deleteComponent`
  Required: index
  Description: Delete a component based on the chosen index. Keep in mind that index numbers start at 0 (zero).
- `-deleteAllComponents`
  Description: Delete all components.
- `-importFlightLog`
  Required: flFileName
  Optional: params.xml
  Description: Import a trajectory file using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Import Trajectory dialog.
- `-importGroundControlPoints`
  Required: gcpFileName
  Optional: params.xml
  Description: Import ground control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Export Ground Control Points dialog.
- `-importControlPointsMeasurements`
  Required: cpmFileName
  Optional: params.xml
  Description: Import measurements of control points (CPs) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Export Control Points Measurements dialog.
- `-editControlPointSelection`
  Required: "key=value"
  Description: Edit the settings/parameters of the selected control points based on the value in the Selected control point(s) panel or its key. More information can be found here.
- `-listControlPoints`
  Required: fileName
  Description: Export a list of control points to the specified file path, including the file name and extension (fileName paramater). Each control point will be listed with its index.
- `-selectControlPoint`
  Required: controlPointName
  Description: Select a control point by its name.
- `-invertControlPointSelection`
  Description: Invert the control point selection. If no control points are selected, all will be selected.
- `-renameControlPoint`
  Required: controlPointName newName
  Description: Rename a control point. Specify control point by its current name (controlPointName) and the new name you want to assign (newName).
- `-renameSelectedControlPoint`
  Required: newName
  Description: Rename the selected control point to the specified new name (newName).
- `-deleteControlPoint`
  Optional: index
  Description: Delete a selected control point. If an optional parameter index is used, a control point with a corresponding index will be removed. Index numbers start at 0 (zero) and follow the order of control points in the 1Ds view.
- `-selectMeasurementByError`
  Required: errorValue
  Optional: controlPointName
  Description: Select any measurement with a position error (in pixels) equal to or greater than the specified errorValue (float value). If no control point is specified (controlPointName), the selection applies to all measurements regardless of their assigned control point.
- `-selectMeasurementByIndex`
  Required: controlPointName index
  Description: Select a control point measurement by its index within the specified control point.
- `-deleteControlPointMeasurement`
  Description: Remove selected control point measurements (images assigned to a control point). Images have to be selected in the 1Ds view under the corresponding control point.
- `-exportGroundControlPoints`
  Required: gcpFileName
  Optional: params.xml
  Description: Export ground control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Export Ground Control dialog.
- `-exportControlPointsMeasurements`
  Required: cpmFileName
  Optional: params.xml
  Description: Export measurements of control points using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Export Control Points Measurements dialog when pressing the Shift key together with the button Control Points in the Export part of the Alignment tab.
- `-defineDistance`
  Required: PointNameA PointNameB distance
  Optional: constraintName
  Description: Define a distance constraint between two control points. If constraintName is not defined, the distance name is created automatically. Alternatively, you can load distance constraints from a file (see below).
- `-defineDistance`
  Required: fileName
  Optional: params.xml
  Description: Import a distance constraint from a file (filename including path) using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the application in the Import Distance Definitions dialog. The list of supported formats distancedefinitions.xml can be found in the installation folder.
- `-editConstraintSelection`
  Required: "key=value"
  Description: Edit the settings/parameters of the selected constraints based on the value in the Selected constraint(s) panel or its key. More information can be found here.
- `-deleteConstraint`
  Optional: index
  Description: Remove the selected distance constraints. The constraints must be selected in the 1Ds view. You can use an index as an optional parameter to specify which constraint to remove. Indexes start at 0 and follow the order of constraints in the 1Ds view.
- `-detectMarkers`
  Optional: params.xml
  Description: Detect markers in images using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the Detect Markers tool in the application.
- `-setCamerasGravityDirection`
  Optional: componentID
  Description: If the image's XMP file contains gravity information (xcr:Gravity), the component will be rotated so the -z vector is in the direction of the gravity vector. This does not apply to the dense point cloud (mesh/model), only to the sparse point cloud (alignment). Will apply to the selected component or to the component defined with the optional parameters (component ID).

### Reconstruction

- `-resetGround`
  Description: Set the ground plane back to its original orientation and position.
- `-setGroundPlaneFromReconstructionRegion`
  Description: Automatically center a model using a reconstruction region into the middle of the grid, adjusting both rotation and transformation.
- `-setReconstructionRegionAuto`
  Description: Set a reconstruction region automatically.
- `-setReconstructionRegion`
  Required: box.rsbox
  Description: Import a reconstruction region from the box.rsbox file.
- `-setReconstructionRegionOnCPs`
  Required: controlPoint | controlPoint | controlPoint | controlPoint|heightValue
  Description: Set a reconstruction region on existing control points. Three control points define the base of the region, and the height of the region is defined by the fourth control point or by entering the height value. You can find more about this command in the examples here.
- `-setReconstructionRegionByDensity`
  Description: Set the reconstruction region to the part of the sparse point cloud with the highest density.
- `-scaleReconstructionRegion`
  Required: scaleX | scaleY | scaleZ
  Optional: origin|center absolute|factor
  Description: Scale the reconstruction region in each axis (scaleX, scaleY, and scaleZ). Scale parameters can be treated either as absolute values (absolute) or scale factors (factor) from the center of the region (center) or its origin (origin) defined by the first control point when using setReconstructionRegionOnCPs command. Default values are "absolute" and "center". You can find more about this command in the examples here.
- `-moveReconstructionRegion`
  Required: moveX | moveY | moveZ
  Description: Move the reconstruction region along the region's axes. The values are in the coordinate system's units. You can find more about this command in the examples here.
- `-rotateReconstructionRegion`
  Required: rotateX | rotateY | rotateZ
  Description: Rotate the reconstruction region around its axes. All values are in degrees. You can find more about this command in the examples here.
- `-offsetReconstructionRegion`
  Required: offsetX | offsetY | offsetZ
  Description: Offset the reconstruction region on its axes by the values of its dimensions. You can find more about this command in the examples here.
- `-exportReconstructionRegion`
  Required: box.rsbox
  Description: Export a reconstruction region to the box.rsbox file.
- `-calculatePreviewModel`
  Description: Calculate 3D mesh in the preview quality.
- `-calculateNormalModel`
  Description: Calculate 3D mesh in the normal quality.
- `-calculateHighModel`
  Description: Calculate 3D mesh in the highest quality.
- `-continueModelCalculation`
  Description: If the model calculation was paused or if there was a crash, it is possible to continue the calculation using this command. Hides the user interface from startup.

### Model Tools

- `-selectModel`
  Required: modelName
  Description: Select a model with the specified name (modelName).
- `-deleteSelectedModel`
  Description: Delete the currently selected model.
- `-duplicateSelectedModel`
  Description: Duplicate the selected model (including textures).
- `-renameSelectedModel`
  Required: newModelName
  Description: Rename the currently selected model.
- `-correctColors`
  Optional: layerName
  Description: Run color correction for all layers or a specified layer (layerName) in the selected component.
- `-unwrap`
  Optional: params.xml
  Description: Calculate the unwrap of a model using the current settings or the settings from the params.xml (optional parameter). You can export these settings from the Unwrap tool in the application.
- `-calculateTexture`
  Optional: params.xml
  Description: Calculate texture using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the Color and Texture Settings panel in the application.
- `-calculateQualityTexture`
  Description: Calculate texture using mesh quality values which can be calculated using the Quality Analysis tool. The values don't have to be calculated prior to using this command.
- `-reprojectTexture`
  Required: sourceModel resultModel
  Optional: params.xml
  Description: Reproject texture from a textured model (sourceModel) to an unwrapped model (resultModel) using the current settings or the settings from the params.xml file (optional parameter). sourceModel and resultModel are model names from the application. You can export these settings from the Texture Reprojection tool in the application.
- `-calculateVertexColors`
  Description: Calculate coloring using the current settings.
- `-calculatePreviewVertexColors`
  Description: Calculate draft coloring using the current settings.
- `-calculateQualityColors`
  Description: Calculate vertex colors using mesh quality values which can be calculated using the Quality Analysis tool. The values don't have to be calculated prior to using this command.
- `-simplify`
  Optional: targetTriangleCount or params.xml
  Description: Simplify a selected model using the current settings (use it without any parameters) to the selected triangle count (targetTriangleCount) or using the settings from the params.xml file. You can export these settings from the Simplify tool in the application.
- `-smooth`
  Optional: params.xml
  Description: Smooth a selected model using the current settings or the settings from the params.xml file. You can export these settings from the Smooth tool in the application.
- `-closeHoles`
  Optional: maxEdgesCount
  Description: Close model holes using the current settings or a specified maximal number of edges (maxEdgesCount).
- `-cleanModel`
  Description: Clean the selected model: remove non-manifold edges and vertices, close small holes, etc.
- `-selectTrianglesInsideReconReg`
  Description: Select triangles inside the reconstruction region.
- `-selectTrianglesOutsideReconReg`
  Description: Select triangles outside the reconstruction region.
- `-selectMarginalTriangles`
  Description: Select triangles that enclose the volume but are not part of the current reconstruction.
- `-selectLargeTrianglesAbs`
  Required: edgeSizeThreshold
  Description: Select triangles with edge lengths larger than the threshold (edgeSizeThreshold).
- `-selectLargeTrianglesRel`
  Required: edgeSizeThreshold
  Description: Select triangles with an edge length larger than the threshold (edgeSizeThreshold) multiplied by the average edge length.
- `-selectLargestModelComponent`
  Description: Select triangles that belong to the largest connected component of the model.
- `-invertTrianglesSelection`
  Description: Invert a selection of triangles.
- `-deselectModelTriangles`
  Description: Deselect all model triangles.
- `-removeSelectedTriangles`
  Description: Create a new model with selected triangles left out (the same behavior as achieved with the Filter Selection tool).
- `-cutByBox`
  Required: inner|outer
  Optional: fillHoles
  Description: Filter out the triangles inside or outside of the reconstruction region. The required parameter defines which triangles will be filtered out, and the optional parameter fillHoles determines if the cut holes will be filled (can be set to true or false, corresponding to the Yes and No values). If the optional parameter is not chosen, the default value will be used (Yes). This tool creates a new model.
- `-undercut`
  Description: Undercut the selected model so that each part contains geometry just in its cluster box.
- `-exportModel`
  Required: modelName fileName
  Optional: params.xml
  Description: Export a model (modelName from the project) as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export model dialog.
- `-exportSelectedModel`
  Required: fileName
  Optional: params.xml
  Description: Export the selected model as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export model dialog.
- `-exportModelToZip`
  Required: filePath
  Optional: modelFormat
  Description: Export the selected model into a compressed archive. Specify the path where the archive will be saved and the name of the exported file. Including the file extension for the archive format is optional. Additionally, you can specify the model format (e.g., .obj or .fbx) as an optional parameter.
- `-importModel`
  Required: fileName
  Optional: params.xml
  Description: Import a model from a file. Specify the full path to the model, as well as its name and format extension. Additionally, use the settings file (exported from the import dialog) to adjust the import settings.
- `-calculateOrthoProjection`
  Optional: rsorthoFile rsboxFile
  Description: Calculate an orthographic projection using the current settings. You can optionally use a projection parameters file (.rsortho) to define the calculation settings. This file can be exported together with an ortho projection, and its structure is described here. An additional optional parameter, the exported reconstruction region file (.rsbox), can be used to specify the area included in the ortho projection.
- `-selectOrthoProjection`
  Required: orthoName
  Description: Select an orthographic projection with the specified name (orthoName).
- `-editOrthoProjectionSelection`
  Required: "key=value"
  Description: Edit the settings/parameters of the selected ortho projections based on the value in the Selected ortho photo(s) panel or its key. More information can be found here.
- `-exportOrthoProjection`
  Required: orthoName fullPath params.xml
  Description: Export the orthographic projection (orthophoto, DSM, or DTM) using settings from the params.xml file, which can be exported from the corresponding export dialog. The command can be written in three ways, depending on which parameters are used. The params.xml file defines the export settings; orthoName is the name of the projection as it appears in the project; fullPath specifies the complete path to the exported file; folderPath defines the directory where the file will be saved; and exportName sets the name of the exported file. For fullPath and exportName, the format extension is optional-if omitted, a TIFF file is created.
- `-calculateCrossSections`
  Optional: step axis
  Description: Calculate Cross Sections either using current settings or by setting the axis (local axis of reconstruction region) and the step between each cross section.
- `-exportCrossSections`
  Required: fileName
  Optional: params.xml
  Description: Export selected cross sections as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export Cross Sections dialog.
- `-renameCrossSections`
  Required: crossSectionsName
  Description: Rename selected cross sections.
- `-selectCrossSections`
  Required: crossSectionsName
  Description: Select cross sections by name.
- `-computeContours`
  Optional: params.xml
  Description: Compute contours for the selected Ortho using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the Contours Tool in the application.
- `-exportContours`
  Required: fileName
  Optional: params.xml
  Description: Export selected contours as a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export Contours dialog.
- `-renameContours`
  Required: contoursName
  Description: Rename selected contours.
- `-selectContours`
  Required: contoursName
  Description: Select contours by name.
- `-exportShapes`
  Required: fileName
  Optional: params.xml
  Description: Export selected shapes as a file (fileName including path and file extension .json) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export Shapes dialog.
- `-importShapesToSelectedOrtho`
  Required: fileName mosaicing|measurements
  Description: Import shapes from a file (fileName including path and file extension) to the selected ortho. Their type will be defined based on which shape-creating tool is activated: Measure or Enhance Mosaic.
- `-importShapesToOrtho`
  Required: fileName orthoProjectionName mosaicing|measurements
  Description: Import shapes from a file (fileName including path and file extension) to ortho (orthoProjectionName). Their type will be defined based on which shape-creating tool is activated: Measure or Enhance Mosaic.
- `-selectShape`
  Required: shapeName
  Description: Select shape by name
- `-addShapeToSelection`
  Required: shapeName
  Description: Add a shape to the current selection by name.
- `-exportReport`
  Required: outputFileName templateFileName
  Optional: true|false
  Description: Export a report into a file (outputFileName including the path and the .html extension) using a template (templateFileName including the path). The default templates are stored in the installation folder\Reports. Use the optional boolean paramater (true or false) to export a file with reports found in the specified template.
- `-printReport`
  Required: reportString
  Description: Write out report texts in the Command Prompt. This command can be used when RealityScan is run with a batch file. It does not work with delegation. Learn more about report customization here.
- `-generateMaskFromMesh`
  Description: Generate mask images out of the existing camera views (images) and selected model. Everything around the model as seen from the camera will be masked out.
- `-exportMapsAndMask`
  Optional: folderName params.xml
  Description: Export masks generated from the camera view over the model, along with depth and normal maps for the selected images. If folderName and params.xml are not specified, the results are saved alongside the original images using the current settings. The parameters file can be exported from the export dialog for maps and masks.
- `-exportLod`
  Required: fileName
  Optional: params.xml
  Description: Export the selected model as a linear Level-of-Detail model into a file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export LoD dialog. For export to Cesium 3D Tiles (.json) file format, use the command export3dTiles.
- `-export3dTiles`
  Required: fileName
  Optional: params.xml
  Description: Export the selected model as a hierarchical Level-of-Detail model into a .json file (fileName including path and file extension) using the current settings or the settings from the params.xml (optional parameter). You can export the file from the Export LoD dialog after selecting Cesium 3D Tiles (.json) file format.
- `-exportCameraSnapshots`
  Required: folderName
  Optional: params.xml
  Description: Render images of a model from the camera positions using all images if none or just one is selected, or using only the selected images. Specify the path of a folder where images will be stored, and use the optional file with parameters to use settings from the export dialog.
- `-exportSelectedCamerasSnapshots`
  Required: folderName fileFormat
  Optional: params.xml
  Description: Render images of a model from the selected camera positions. Specify the path of a folder where images will be stored and the file format of the exported images (e.g. jpg or png). Use the optional file with parameters to use settings from the export dialog.
- `-renderMeshFromCustomPositionYPR`
  Required: fileName params.xml|fileName width height focalLength x y z yaw pitch roll
  Description: Export a render of the model from a camera position with the viewpoint defined by yaw, pitch, and roll orientation. | Example for a render from above of a model at 0,0,0 in local Euclidean space: RealityScan.exe -renderMeshFromCustomPositionYPR "D:/Project/render.png" 1280 720 100 0 0 150 0 0 0
- `-renderMeshFromCustomPositionLookAt`
  Required: fileName params.xml|fileName width height focalLength x y z atX atY atZ
  Optional: upX upY upZ
  Description: Export a render of a model from a camera position, which looks at another defined position, with an option to specify the vertical axis. | Example for a side view of a model at 0,0,0 in local Euclidean space while defining the vertical axis going up the Z axis: RealityScan.exe -renderMeshFromCustomPositionLookAt "D:/Project/render.png" 1280 720 50 0 -100 0 0 0 10 0 0 1
- `-renderMeshFromCustomGridPositionYPR`
  Required: fileName params.xml|fileName width height focalLength x y z yaw pitch roll
  Description: Export a render of the model from a custom grid position, with the viewpoint defined by yaw, pitch, and roll orientation.
- `-renderMeshFromCustomGridPositionLookAt`
  Required: fileName params.xml|fileName width height focalLength x y z atX atY atZ
  Optional: upX upY upZ
  Description: Export a model render from a custom grid position, with the viewpoint directed toward a defined target, and the option to specify the vertical axis.
- `-uploadToSketchfab`
  Required: APIToken
  Description: Upload your model to Sketchfab. The required parameter is the Sketchfab API token and it can be found in the account settings of your Sketchfab account.
- `-dtmClassify`
  Optional: params.xml
  Description: Classify vertices of the selected model into pre-defined classes using the current settings or the settings from the params.xml file (optional parameter). You can export these settings from the Classify tool in the application.
- `-selectClassification`
  Required: classificationName
  Description: Select a classification using its specified name (classificationName parameter).
- `-renameSelectedClassification`
  Required: newClassificationName
  Description: Rename the selected classification to the specified new name (newClassificationName).
- `-transferClassification`
  Optional: params.xml
  Description: Transfer classification from the labels layer images. A file with parameters exported from the AI Classify tool panel can be used as an optional parameter.
- `-exportClassificationSettings`
  Required: XMLfilePath
  Description: Export settings from the AI Classify tool panel. For the required parameter (XMLfilePath) specify the full path for export, including the file name and format extension (.xml).
- `-importClassificationSettings`
  Required: XMLfilePath
  Description: Import settings to the AI Classify tool panel. For the required parameter (XMLfilePath) specify the full path for import, including the file name and format extension (.xml).
- `-overrideSelectedVertices`
  Optional: className
  Description: Override the selected vertices and change their class to the currently selected class, or to the class specified by name (optional).
- `-selectClass`
  Required: className
  Description: Select a class from the selected classification using the specified class name.
- `-deselectClass`
  Description: Deselect all selected classes.
- `-renameSelectedClass`
  Required: newClassName
  Description: Change the name of the selected class to the specified new class name (newClassName).
- `-setSelectedClassAsGroundForDTM`
  Required: true OR false
  Description: Change the Use as ground for DTM setting in the Selected Class panel for the selected class. Set to true to use it as ground for DTM, or false to exclude it.
- `-setSelectedClassAsGroundForExport`
  Description: Change the Export class LAS setting in the Selected Class panel to Ground (2).
- `-setSelectedClassLasFormat`
  Required: 0 - 12
  Description: Change the Export class LAS setting to a value corresponding to a number from 0 to 12.
- `-selectVerticesOfSelectedClass`
  Description: Select model vertices based on the selected class.
- `-selectClassificationFormat`
  Required: classificationFormatName
  Description: Select the classification format based on the specified classification format name.
- `-renameSelectedClassificationFormat`
  Required: newClassificationFormatName
  Description: Rename the selected classification format to the specified new classification format name.
- `-exportSelectedClassificationFormat`
  Required: filePath
  Description: Export the selected classification format by specifying the full path, including the file name and the .cfd extension.
- `-exportClassificationFormat`
  Required: classificationFormatName filePath
  Description: Export the classification format by specifying its name and the output file path, including the file name and the .cfd extension.
- `-importClassificationFormat`
  Required: filePath
  Description: Import the classification format by specifying its full path and name, including the .cfd extension.
- `-colorModelBySelectedClassification`
  Description: Colorize the model according to the classes in the selected classification.
- `-deleteSelectedClassification`
  Description: Delete the selected classification.

### Settings' and Error-handling Commands

- `-set`
  Required: "key=value"
  Description: Change an application setting. Learn more about how to use this command here.
- `-preset`
  Required: "key=value"
  Description: Change an application setting during the setup phase. Ideal for changes that require a reset of the application. Learn more about how to use this command here.
- `-reset`
  Required: ui | cfg | cfgui | all
  Description: Reset the user interface, settings, or both. It is also possible to make RealityScan like a clean install. This command works only when used in a batch file, and it won't work with delegation commands. What is going to be reset is determined by the chosen parameter: ui - reset user interface, cfg - reset application settings, cfgui - reset both interface and settings, all - make it like a clean install.
- `-silent`
  Required: crashReportPath
  Description: Suppress warning dialogs and uploading of the crash reports. The application will store reports in the specified location instead of showing the upload wizard. This command has to be used at the startup.
- `-writeProgress`
  Required: fileName
  Optional: timeout
  Description: Write any new progress change into a specified file (fileName including the path). Optional timeout parameter will also output during a defined period of time (timeout in seconds). More about the file structure can be found in the section Error-handling Commands.
- `-printProgress`
  Optional: timeout
  Description: Print progress change into the Windows Command Prompt for any new change. Optional timeout parameter will also output during a defined period of time (timeout in seconds).
- `-tag`
  Description: Writes out a tag into the Windows Command Prompt. It respects the order of the used commands and will be run after the process ran before it finishes.
- `-stdConsole`
  Description: Enables console redirection to the application standard output. When used, you will see the application console content mirrored also in the standard Windows console. This also enables further redirections for CLI purposes.
- `-disableOnlineCommunication`
  Description: Disable any online communication.
- `-importGlobalSettings`
  Required: settings.rcconfig
  Description: Import application global settings from the settings.rcconfig file.
- `-exportGlobalSettings`
  Required: settings.rcconfig
  Description: Export application global settings to the settings.rcconfig file.
- `-setProjectCoordinateSystem`
  Required: authority:id
  Description: Set a project coordinate system defined by an authority, and its ID (can be found in a specific database, e.g. epsg.xml and local.xml). An example can be found here.
- `-setOutputCoordinateSystem`
  Required: authority:id
  Description: Set an output coordinate system defined by an authority, and its ID (can be found in a specific database, e.g. epsg.xml and local.xml). An example can be found here.

## Source Coverage and Verification Notes

Source HTML pages used for the main rebuild:

- `List of All CLI Commands - RealityScan Help.html` as the master command inventory.
- `Command Line Interface - RealityScan Help.html` for workflow framing and basic usage examples.
- `Alignment Commands - RealityScan Help.html` for alignment, component, and control-point workflows.
- `Reconstruction Commands - RealityScan Help.html` for reconstruction-region and model-calculation workflows.
- `Model Tools - RealityScan Help.html` for post-processing and export workflows.
- `Settings' Commands - RealityScan Help.html` for application settings behavior.
- `Error-handling Commands - RealityScan Help.html` for unattended execution and failure handling.
- `Commands Outside Command Prompt - RealityScan Help.html` for command-file behavior.
- `Delegation of Commands - RealityScan Help.html` for multi-instance orchestration.
- `Parametric Files of Orthographic Projections - RealityScan Help.html` for `.rcortho` usage.
- `Process IDs - RealityScan Help.html` for progress-monitoring context.

Verification summary:

- Master command inventory count: 209
- All commands from the master list are included in the appendix above.
- Commands are normalized to markdown CLI form with a leading `-`.
- Repeated website navigation blocks, `See also` fragments, and orphaned web-link fragments were intentionally excluded.
- Non-CLI pages were reviewed and intentionally reduced in scope so the file stays focused on agent CLI operation.

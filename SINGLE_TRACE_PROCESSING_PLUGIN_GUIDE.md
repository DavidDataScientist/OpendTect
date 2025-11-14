# Step-by-Step Guide to Single Trace Processing Plugin Development

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Plugin Architecture](#plugin-architecture)
4. [Step 1: Project Structure](#step-1-project-structure)
5. [Step 2: Create the Backend Module](#step-2-create-the-backend-module)
6. [Step 3: Create the UI Module](#step-3-create-the-ui-module)
7. [Step 4: Build Configuration](#step-4-build-configuration)
8. [Step 5: Building and Testing](#step-5-building-and-testing)
9. [Complete Example](#complete-example)

---

## Overview

A **single trace processing plugin** in OpendTect is an attribute that processes seismic traces independently - each output trace is computed solely from the corresponding input trace(s) without requiring neighboring traces (though you can optionally use steering data or apply smoothing filters that access neighboring traces).

Common examples:
- **Scaling/Shifting**: Multiply trace values by a factor and add an offset
- **Mathematical operations**: Square, absolute value, logarithm
- **Simple filters**: Running average along the trace

This guide will teach you how to create a plugin that scales and shifts seismic traces, similar to the Tutorial attribute in the codebase.

---

## Prerequisites

Before starting, ensure you have:
- OpendTect source code properly set up
- C++ compiler (gcc 10.3+, MSVC 2022, or Xcode)
- CMake 3.24 or higher
- Qt (version depends on your OpendTect branch)
- Basic understanding of C++ and CMake

---

## Plugin Architecture

An OpendTect attribute plugin consists of **two modules**:

### 1. Backend Module (`MyPlugin`)
Contains the core processing logic:
- **Attribute Provider class**: Inherits from `Attrib::Provider`
- **Plugin initialization**: Registers the attribute with OpendTect
- Can be loaded into batch processing tools

### 2. UI Module (`uiMyPlugin`)
Contains the user interface:
- **UI class**: Inherits from `uiAttrDescEd`
- **Plugin initialization**: Registers the UI with OpendTect
- Only loaded into the main GUI application

---

## Step 1: Project Structure

Create the following directory structure in `plugins/`:

```
plugins/
├── MyScaler/                    # Backend module
│   ├── CMakeLists.txt
│   ├── myscalerattrib.h
│   ├── myscalerattrib.cc
│   └── myscalerpi.cc           # Plugin initialization
│
└── uiMyScaler/                  # UI module
    ├── CMakeLists.txt
    ├── uimyscalerattrib.h
    ├── uimyscalerattrib.cc
    └── uimyscalerpi.cc          # Plugin initialization
```

---

## Step 2: Create the Backend Module

### 2.1 Header File: `myscalerattrib.h`

Create the attribute provider class:

```cpp
// =============================================================================
// HEADER FILE: myscalerattrib.h
// This file declares the MyScaler attribute class
// =============================================================================

// #pragma once ensures this header file is only included once during compilation
// This prevents "multiple definition" errors
#pragma once

// Include the base class Provider that all OpendTect attributes inherit from
// This gives us access to methods like getInputValue(), setOutputValue(), etc.
#include "attribprovider.h"

// All OpendTect attributes must be in the Attrib namespace
// This helps organize code and avoid name conflicts with other libraries
namespace Attrib
{

/*!\brief MyScaler Attribute
 *
 * This comment block documents what the attribute does.
 * It's good practice to list all inputs and outputs here.
 *
 * Scales and shifts seismic trace values using the formula:
 * output = (input * factor) + shift
 *
 * Input:
 * 0       Input seismic data (this is the first and only input)
 *
 * Outputs:
 * 0       Scaled and shifted trace (this is the first and only output)
 */

// Our attribute class inherits from Provider (the base class for all attributes)
// "public" means we inherit all public and protected methods from Provider
class MyScaler : public Provider
{
public:
    // PUBLIC SECTION: These methods can be called from anywhere
    
    // initClass() is called when OpendTect starts up
    // It registers this attribute with OpendTect's factory system
    // MUST be static (belongs to the class, not any instance)
    static void         initClass();
    
    // Constructor: Creates a new MyScaler object
    // Takes a Desc& (description) that contains all the parameter values
    // The & means "reference" - we're not copying the Desc, just referring to it
                        MyScaler(Desc&);
    
    // -------------------------------------------------------------------------
    // These static methods return strings that identify parameters
    // They MUST return the same strings in both .h and .cc files
    // OpendTect uses these strings internally to store/retrieve parameter values
    // -------------------------------------------------------------------------
    
    // attribName() returns "MyScaler" - this is how users see your attribute
    // This name appears in OpendTect's attribute selection menu
    static const char*  attribName()    { return "MyScaler"; }
    
    // factorStr() returns "factor" - the internal name for the multiplication factor
    // Used to retrieve the factor value from the parameter storage
    static const char*  factorStr()     { return "factor"; }
    
    // shiftStr() returns "shift" - the internal name for the shift value
    // Used to retrieve the shift value from the parameter storage
    static const char*  shiftStr()      { return "shift"; }

protected:
    // PROTECTED SECTION: These can only be called by this class and derived classes
    
    // Destructor: Cleans up when a MyScaler object is destroyed
    // Virtual means derived classes can override it
    // Protected (not public) because only OpendTect should delete these objects
                        ~MyScaler();
    
    // createInstance() is called by OpendTect's factory to create new MyScaler objects
    // Returns a Provider pointer (base class) so factory doesn't need to know about MyScaler
    // MUST be static and MUST be named exactly "createInstance"
    static Provider*    createInstance(Desc&);
    
    // -------------------------------------------------------------------------
    // CRITICAL METHODS: These control how your attribute executes
    // -------------------------------------------------------------------------
    
    // allowParallelComputation() tells OpendTect if this attribute is thread-safe
    // Return true if your attribute doesn't use shared/global variables
    // Single trace processing is almost always thread-safe, so return true
    // This allows multiple traces to be processed simultaneously for speed
    // "override" means we're replacing the base class implementation
    // "const" means this method doesn't modify the object
    bool                allowParallelComputation() const override
                                                        { return true; }
    
    // getInputData() is called once per trace before computeData()
    // It retrieves the input data needed for processing
    // Parameters:
    //   - BinID: The inline/crossline position of the trace
    //   - zintv: The depth/time interval to retrieve
    // Returns: true if data was retrieved successfully, false otherwise
    bool                getInputData(const BinID&, int zintv) override;
    
    // computeData() is THE MAIN METHOD - this is where you do your processing!
    // It's called for each trace and computes the output values
    // Parameters:
    //   - output: DataHolder where you write your results
    //   - relpos: Relative position (usually 0,0 for single trace processing)
    //   - z0: Starting depth/time sample index
    //   - nrsamples: Number of samples to process in this call
    //   - threadid: ID of the thread (useful for debugging parallel execution)
    // Returns: true if processing succeeded, false if there was an error
    // "const" means this method doesn't modify member variables (thread-safe!)
    bool                computeData(const DataHolder& output, 
                                   const BinID& relpos,
                                   int z0, int nrsamples, 
                                   int threadid) const override;
    
    // -------------------------------------------------------------------------
    // MEMBER VARIABLES: These store the state of each MyScaler object
    // -------------------------------------------------------------------------
    
    // Parameter values (set in constructor from the Desc)
    float               factor_;      // Multiplication factor (e.g., 2.0 doubles amplitudes)
    float               shift_;       // Value to add after multiplication (e.g., 100)
    
    // Input data storage (set in getInputData(), used in computeData())
    const DataHolder*   inputdata_;   // Pointer to the input trace data
                                      // "const" means we won't modify the input
                                      // Pointer can be null, so always check before using!
    
    int                 dataidx_;     // Index for accessing the correct input
                                      // Each input has an index (0 for first input, 1 for second, etc.)
                                      // Set to -1 initially to indicate "not set yet"
};

} // namespace Attrib - closes the namespace we opened at the top
```

### 2.2 Implementation File: `myscalerattrib.cc`

Implement the attribute provider:

```cpp
// =============================================================================
// IMPLEMENTATION FILE: myscalerattrib.cc
// This file implements the methods declared in myscalerattrib.h
// =============================================================================

// Include our header file that contains the class declaration
#include "myscalerattrib.h"

// Include OpendTect headers we need
#include "attribdataholder.h"  // DataHolder class for storing trace data
#include "attribdesc.h"         // Desc class for attribute descriptions
#include "attribfactory.h"      // Factory system for creating attributes
#include "attribparam.h"        // Parameter classes (FloatParam, etc.)

// All our code is in the Attrib namespace (must match the .h file)
namespace Attrib
{

// =============================================================================
// FACTORY REGISTRATION
// =============================================================================

// This macro registers our MyScaler class with OpendTect's factory system
// It creates a createInstance() function that OpendTect calls to make new MyScaler objects
// MUST be at file scope (not inside any function)
// The name "MyScaler" MUST match your class name exactly
mAttrDefCreateInstance(MyScaler)

// =============================================================================
// initClass() - ATTRIBUTE INITIALIZATION
// Called once when OpendTect starts (or when plugin is loaded)
// This is where you define ALL parameters, inputs, and outputs
// =============================================================================
void MyScaler::initClass()
{
    // This macro starts the initialization process
    // It creates a "desc" variable (attribute descriptor) that we'll configure
    // MUST be the first line in initClass()
    mAttrStartInitClass
    
    // -------------------------------------------------------------------------
    // DEFINE PARAMETER: factor (multiplication factor)
    // -------------------------------------------------------------------------
    
    // Create a new floating-point parameter
    // factorStr() returns "factor" - this is the internal storage name
    FloatParam* factor = new FloatParam(factorStr());
    
    // Set the default value that appears when user first opens the dialog
    // If user doesn't change it, this value will be used
    factor->setDefaultValue(1.0f);  // 1.0 means no scaling by default
    
    // Set valid range for this parameter
    // User won't be able to enter values outside this range in the UI
    // Interval<float>(min, max) creates a range object
    factor->setLimits(Interval<float>(-1000.0f, 1000.0f));
    
    // Add this parameter to the attribute description
    // Now OpendTect knows this attribute has a "factor" parameter
    desc->addParam(factor);
    
    // -------------------------------------------------------------------------
    // DEFINE PARAMETER: shift (value to add)
    // -------------------------------------------------------------------------
    
    // Create another floating-point parameter for the shift value
    FloatParam* shift = new FloatParam(shiftStr());  // shiftStr() returns "shift"
    
    // Default shift is 0 (no offset)
    shift->setDefaultValue(0.0f);
    
    // Add to description (we don't set limits, so any value is allowed)
    desc->addParam(shift);
    
    // -------------------------------------------------------------------------
    // DEFINE OUTPUT DATA TYPE
    // -------------------------------------------------------------------------
    
    // Tell OpendTect what type of data this attribute produces
    // Seis::UnknowData means it's generic seismic data (most common)
    // Other options: Seis::Ampl (amplitude), Seis::Frequency, etc.
    desc->addOutputDataType(Seis::UnknowData);
    
    // -------------------------------------------------------------------------
    // DEFINE REQUIRED INPUT
    // -------------------------------------------------------------------------
    
    // Specify that this attribute needs one input
    // InputSpec("Input data", true) creates an input specification
    //   - "Input data" is the label shown in the UI
    //   - true means this input is REQUIRED (user must provide it)
    // This will be input #0 (first input)
    desc->addInput(InputSpec("Input data", true));
    
    // This macro ends the initialization
    // It registers the configured "desc" with OpendTect's factory
    // MUST be the last line in initClass()
    mAttrEndInitClass
}

// =============================================================================
// CONSTRUCTOR - Creates a new MyScaler object
// Called by the factory when OpendTect needs to process data with this attribute
// =============================================================================
MyScaler::MyScaler(Desc& desc)
    : Provider(desc)           // Call base class constructor first
    , inputdata_(nullptr)      // Initialize input pointer to null (no data yet)
    , dataidx_(-1)            // Initialize data index to -1 (invalid/not set)
{
    // Check if the base class initialization succeeded
    // isOK() returns false if something went wrong (e.g., invalid parameters)
    if (!isOK())
        return;  // Exit early if there's a problem
    
    // -------------------------------------------------------------------------
    // RETRIEVE PARAMETER VALUES from the description
    // -------------------------------------------------------------------------
    
    // mGetFloat is a macro that retrieves a float parameter
    // Syntax: mGetFloat(variable_to_store_in, parameter_name_function)
    // It reads the "factor" parameter value and stores it in factor_
    mGetFloat(factor_, factorStr());
    
    // Same for shift parameter
    // After these calls, factor_ and shift_ contain the user's chosen values
    mGetFloat(shift_, shiftStr());
    
    // Now factor_ and shift_ are ready to use in computeData()!
}

// =============================================================================
// DESTRUCTOR - Cleans up when MyScaler object is destroyed
// =============================================================================
MyScaler::~MyScaler()
{
    // Nothing to clean up for this simple attribute
    // The base class destructor handles most cleanup automatically
    // If you allocated memory with "new", you would delete it here
}

// =============================================================================
// getInputData() - Retrieves input data for one trace
// Called once per trace BEFORE computeData()
// =============================================================================
bool MyScaler::getInputData(const BinID& relpos, int zintv)
{
    // -------------------------------------------------------------------------
    // GET DATA FROM FIRST INPUT (input #0)
    // -------------------------------------------------------------------------
    
    // inputs_[0] is the first input (the seismic data we're processing)
    // getData(relpos, zintv) retrieves the trace at position relpos for interval zintv
    // Returns a DataHolder pointer, or nullptr if data couldn't be retrieved
    inputdata_ = inputs_[0]->getData(relpos, zintv);
    
    // Check if we got valid data
    if (!inputdata_)
        return false;  // Return false = "I couldn't get the data I need"
                       // OpendTect will skip this trace and maybe try again later
    
    // -------------------------------------------------------------------------
    // GET THE DATA INDEX
    // -------------------------------------------------------------------------
    
    // Each input can have multiple "series" of data
    // getDataIndex(0) returns the index for accessing input #0's data
    // We need this index to retrieve values in computeData()
    dataidx_ = getDataIndex(0);
    
    // Success! We have the input data and know how to access it
    return true;
}

// =============================================================================
// computeData() - THE MAIN PROCESSING FUNCTION
// This is where the actual trace processing happens!
// Called once per trace, possibly multiple times per trace if trace is long
// =============================================================================
bool MyScaler::computeData(const DataHolder& output,    // Where to write results
                          const BinID& relpos,          // Position (often 0,0)
                          int z0,                       // Starting sample index
                          int nrsamples,                // Number of samples to process
                          int threadid) const           // Thread ID (for parallel processing)
{
    // -------------------------------------------------------------------------
    // SAFETY CHECK
    // -------------------------------------------------------------------------
    
    // Make sure we have input data
    // If getInputData() failed, inputdata_ will be nullptr
    if (!inputdata_)
        return false;  // Can't process without input data!
    
    // -------------------------------------------------------------------------
    // PROCESS EACH SAMPLE IN THE TRACE
    // -------------------------------------------------------------------------
    
    // Loop through all samples we need to compute
    // idx is relative to z0 (idx=0 means sample at position z0)
    for (int idx = 0; idx < nrsamples; idx++)
    {
        // ---------------------------------------------------------------------
        // STEP 1: Get the input value at this sample
        // ---------------------------------------------------------------------
        
        // getInputValue() retrieves one sample from the input trace
        // Parameters:
        //   *inputdata_ - the input data (dereference the pointer with *)
        //   dataidx_    - which input series to read from
        //   idx         - sample index relative to z0
        //   z0          - starting position
        // Returns: the seismic amplitude value at this position
        const float inval = getInputValue(*inputdata_, dataidx_, idx, z0);
        
        // ---------------------------------------------------------------------
        // STEP 2: Apply our algorithm (scale and shift)
        // ---------------------------------------------------------------------
        
        // This is the heart of our attribute!
        // Formula: output = (input * factor) + shift
        // Examples:
        //   - factor=2.0, shift=0:    doubles all values
        //   - factor=1.0, shift=100:  adds 100 to all values
        //   - factor=0.5, shift=0:    halves all values
        const float outval = inval * factor_ + shift_;
        
        // ---------------------------------------------------------------------
        // STEP 3: Store the result in the output
        // ---------------------------------------------------------------------
        
        // setOutputValue() writes one sample to the output trace
        // Parameters:
        //   output - the output DataHolder
        //   0      - output series index (0 = first output)
        //   idx    - sample index relative to z0
        //   z0     - starting position
        //   outval - the value to write
        setOutputValue(output, 0, idx, z0, outval);
    }
    
    // Success! All samples were processed
    return true;
}

} // namespace Attrib - close the namespace
```

### 2.3 Plugin Initialization: `myscalerpi.cc`

Register the plugin with OpendTect:

```cpp
// =============================================================================
// PLUGIN INITIALIZATION FILE: myscalerpi.cc
// This file tells OpendTect about your plugin
// "pi" stands for "plugin init"
// =============================================================================

// Include our attribute class
#include "myscalerattrib.h"

// Include OpendTect's plugin system
#include "odplugin.h"

// =============================================================================
// PLUGIN INFORMATION
// This tells OpendTect about your plugin (name, version, description, etc.)
// =============================================================================

// mDefODPluginInfo is a macro that creates the getODPluginInfo() function
// OpendTect calls this function to learn about your plugin
// The parameter "MyScaler" MUST match your module name
mDefODPluginInfo(MyScaler)
{
    // Create a PluginInfo object with all the metadata
    // "static" means this object persists - OpendTect keeps the pointer
    static PluginInfo retpi(
        // Display name - shown in plugin manager and about dialog
        "MyScaler Plugin (Base)",
        
        // Product name - usually "OpendTect" for free plugins
        "OpendTect",
        
        // Creator/vendor name - your name or company
        "Your Name",
        
        // Version string - use semantic versioning (major.minor.patch)
        "1.0.0",
        
        // Description - shown in plugin manager
        // Use \n for line breaks
        "Single trace scaling and shifting attribute.\n"
        "This module can be loaded into batch processing tools.");
    
    // Return pointer to the PluginInfo object
    // OpendTect stores this and displays it in the UI
    return &retpi;
}

// =============================================================================
// PLUGIN INITIALIZATION FUNCTION
// This is called when OpendTect loads your plugin
// =============================================================================

// mDefODInitPlugin creates the initODPlugin() function
// OpendTect calls this when loading the plugin
// The parameter "MyScaler" MUST match your module name
mDefODInitPlugin(MyScaler)
{
    // Initialize our attribute class
    // This calls MyScaler::initClass() which registers the attribute
    // The Attrib:: namespace prefix is needed because we're outside the namespace
    Attrib::MyScaler::initClass();
    
    // Return nullptr to indicate success
    // If initialization failed, you could return an error message:
    //   return "Failed to initialize MyScaler: reason here";
    // But for simple plugins, just return nullptr
    return nullptr;
}

// =============================================================================
// THAT'S IT FOR THE BACKEND!
// At this point you have:
//   1. Defined your attribute class (myscalerattrib.h/cc)
//   2. Registered it with OpendTect (myscalerpi.cc)
//   3. Your attribute can be used in batch processing
// 
// Next: Create the UI module so users can configure it in the GUI
// =============================================================================
```

### 2.4 CMakeLists.txt for Backend

```cmake
# =============================================================================
# CMAKE BUILD CONFIGURATION for MyScaler Backend Module
# This file tells CMake how to build your plugin
# =============================================================================

# -----------------------------------------------------------------------------
# DEPENDENCIES: What other OpendTect modules does this plugin need?
# -----------------------------------------------------------------------------

# This plugin needs the "Attributes" module (provides Provider, Desc, etc.)
# If your plugin needs other modules, add them here separated by spaces
# Example: set(OD_MODULE_DEPS Attributes Network)
set(OD_MODULE_DEPS Attributes)

# -----------------------------------------------------------------------------
# ORGANIZATION: Where should this plugin appear in the IDE?
# -----------------------------------------------------------------------------

# This is just for organizing plugins in Visual Studio, Xcode, etc.
# All plugins in "My Plugins" folder will be grouped together
set(OD_FOLDER "My Plugins")

# -----------------------------------------------------------------------------
# PLUGIN FLAG: Tell OpendTect this is a plugin (not a core module)
# -----------------------------------------------------------------------------

# MUST be "yes" for plugins
# This affects how the module is built and loaded
set(OD_IS_PLUGIN yes)

# -----------------------------------------------------------------------------
# SOURCE FILES: What .cc files should be compiled?
# -----------------------------------------------------------------------------

# List all .cc implementation files
# Do NOT list .h header files here - CMake finds them automatically
set(OD_MODULE_SOURCES
    myscalerattrib.cc    # Main attribute implementation
    myscalerpi.cc        # Plugin initialization
)

# Note: If you add more source files later, add them to this list

# -----------------------------------------------------------------------------
# PLUGIN EXECUTABLES: Which OpendTect programs should load this plugin?
# -----------------------------------------------------------------------------

# ${OD_ATTRIB_EXECS} is a variable containing all attribute processing programs:
#   - od_process_attrib (batch attribute processing)
#   - od_process_attrib_em (batch EM attribute processing)
#   - Any other programs that work with attributes
# This plugin will be automatically loaded by these programs
set(OD_PLUGIN_ALO_EXEC
    ${OD_ATTRIB_EXECS}
)

# Note: ALO = "Auto-Load" - the plugin loads automatically when these programs start

# -----------------------------------------------------------------------------
# INITIALIZE MODULE: Run OpendTect's build system
# -----------------------------------------------------------------------------

# This macro does all the heavy lifting:
#   - Creates the build target
#   - Links dependencies
#   - Sets up include paths
#   - Configures installation
# MUST be called at the end of every OpendTect CMakeLists.txt
OD_INIT_MODULE()
```

---

## Step 3: Create the UI Module

### 3.1 Header File: `uimyscalerattrib.h`

Create the UI class:

```cpp
// =============================================================================
// UI HEADER FILE: uimyscalerattrib.h
// This file declares the UI class for configuring the MyScaler attribute
// =============================================================================

// Prevent multiple inclusion
#pragma once

// Include the base class for attribute UI dialogs
#include "uiattrdesced.h"

// Forward declarations - tell compiler these classes exist without including headers
// This speeds up compilation and avoids circular dependencies
namespace Attrib { class Desc; }  // The attribute description class (from backend)
class uiAttrSel;                  // Widget for selecting input attributes
class uiGenInput;                 // Generic input widget (can be text, number, dropdown, etc.)

// =============================================================================
// UI CLASS: uiMyScalerAttrib
// This creates the dialog box where users configure the MyScaler attribute
// =============================================================================

// Inherits from uiAttrDescEd - the base class for ALL attribute UI dialogs
// uiAttrDescEd provides:
//   - Standard dialog layout
//   - OK/Cancel buttons
//   - Input selection helpers
//   - Connection to attribute system
class uiMyScalerAttrib : public uiAttrDescEd
{
public:
    // PUBLIC SECTION: Methods that can be called from anywhere
    
    // -------------------------------------------------------------------------
    // CONSTRUCTOR: Creates the dialog
    // -------------------------------------------------------------------------
    // Parameters:
    //   uiParent* - The parent window (usually the main OpendTect window)
    //   bool is2d - true if this is for 2D data, false for 3D data
    //               This affects which input types are available
                        uiMyScalerAttrib(uiParent*, bool is2d);
    
    // -------------------------------------------------------------------------
    // DESTRUCTOR: Cleans up when dialog is closed
    // -------------------------------------------------------------------------
                        ~uiMyScalerAttrib();

protected:
    // PROTECTED SECTION: Methods and variables for internal use
    
    // -------------------------------------------------------------------------
    // UI WIDGETS: These are the actual input fields users interact with
    // -------------------------------------------------------------------------
    
    // Pointer to the input selection dropdown
    // Users use this to choose which seismic data to process
    // NULL until created in constructor
    uiAttrSel*          inpfld_;
    
    // Pointer to the factor input field (numeric entry)
    // Users type the multiplication factor here (e.g., 2.0)
    uiGenInput*         factorfld_;
    
    // Pointer to the shift input field (numeric entry)
    // Users type the shift value here (e.g., 100)
    uiGenInput*         shiftfld_;
    
    // -------------------------------------------------------------------------
    // REQUIRED METHODS: You MUST implement these four methods
    // They handle the bidirectional flow of data between UI and backend
    // -------------------------------------------------------------------------
    
    // setParameters() - Load parameter values FROM Desc INTO UI widgets
    // Called when: User opens an existing attribute for editing
    // Purpose: Populate the dialog with saved values
    // Parameters:
    //   const Attrib::Desc& - The attribute description containing saved values
    //                        ("const" means we won't modify it)
    // Returns: true if successful, false if this isn't our attribute
    bool                setParameters(const Attrib::Desc&) override;
    
    // setInput() - Load input selection FROM Desc INTO UI
    // Called when: User opens an existing attribute for editing
    // Purpose: Set which seismic data was selected as input
    // Parameters:
    //   const Attrib::Desc& - The attribute description
    // Returns: true if successful
    bool                setInput(const Attrib::Desc&) override;
    
    // getParameters() - Save parameter values FROM UI widgets INTO Desc
    // Called when: User clicks OK button
    // Purpose: Store the user's choices in the attribute description
    // Parameters:
    //   Attrib::Desc& - The attribute description to modify
    //                  (no "const" because we're writing to it)
    // Returns: true if values are valid, false if there's an error
    bool                getParameters(Attrib::Desc&) override;
    
    // getInput() - Save input selection FROM UI INTO Desc
    // Called when: User clicks OK button
    // Purpose: Store which seismic data user selected
    // Parameters:
    //   Attrib::Desc& - The attribute description to modify
    // Returns: true if successful
    bool                getInput(Attrib::Desc&) override;
    
    // -------------------------------------------------------------------------
    // MACRO: Declares additional required methods
    // -------------------------------------------------------------------------
    // This macro declares some boilerplate methods needed by the UI system
    // Don't worry about what it does internally - just include it
    mDeclReqAttribUIFns
};
```

### 3.2 Implementation File: `uimyscalerattrib.cc`

Implement the UI:

```cpp
// =============================================================================
// UI IMPLEMENTATION FILE: uimyscalerattrib.cc
// This file implements the UI class declared in uimyscalerattrib.h
// =============================================================================

// Include our UI header
#include "uimyscalerattrib.h"

// Include the backend attribute class (we need to reference it)
#include "myscalerattrib.h"

// Include OpendTect headers we need
#include "attribdesc.h"       // Attribute description class
#include "attribparam.h"      // Parameter access methods
#include "uiattrsel.h"        // Input selection widget
#include "uigeninput.h"       // Generic input widget
#include "uiattribfactory.h"  // UI factory registration

// Use the Attrib namespace so we don't need to type "Attrib::" everywhere
using namespace Attrib;

// =============================================================================
// UI FACTORY REGISTRATION
// This macro registers our UI with OpendTect's factory system
// =============================================================================

// mInitAttribUI registers the UI so OpendTect knows how to create it
// Parameters:
//   1. UI class name: uiMyScalerAttrib
//   2. Backend class name: MyScaler (must match backend)
//   3. Display name: "MyScaler" (shown in menus)
//   4. Group: sKeyBasicGrp() means "Basic" group in attribute menu
//            Other options: sKeyFilterGrp(), sKeyFreqGrp(), etc.
mInitAttribUI(uiMyScalerAttrib, MyScaler, "MyScaler", sKeyBasicGrp())

// =============================================================================
// CONSTRUCTOR - Builds the dialog box
// This is where you create all the UI widgets (input fields, buttons, etc.)
// =============================================================================
uiMyScalerAttrib::uiMyScalerAttrib(uiParent* p, bool is2d)
    : uiAttrDescEd(p, is2d, HelpKey("myscaler", "attrib"))  // Call base constructor
    // Parameters to base constructor:
    //   p - parent window
    //   is2d - whether this is 2D or 3D data
    //   HelpKey - defines help context (topic, key) for F1 help
{
    // -------------------------------------------------------------------------
    // CREATE INPUT SELECTION WIDGET
    // -------------------------------------------------------------------------
    
    // createInpFld() is a helper method from uiAttrDescEd
    // It creates a dropdown where users select which seismic data to process
    // Automatically populated with available seismic data in the project
    // is2d parameter determines if 2D or 3D data is shown
    inpfld_ = createInpFld(is2d);
    
    // -------------------------------------------------------------------------
    // CREATE FACTOR INPUT FIELD
    // -------------------------------------------------------------------------
    
    // Create a numeric input field for the multiplication factor
    // uiGenInput is a flexible widget that can be:
    //   - Text input (StringInpSpec)
    //   - Number input (IntInpSpec, FloatInpSpec)
    //   - Checkbox (BoolInpSpec)
    //   - Dropdown (StringListInpSpec)
    //
    // Parameters:
    //   this - parent widget (this dialog)
    //   tr("Multiplication Factor") - label shown next to the field
    //                                tr() = "translate" for internationalization
    //   FloatInpSpec(1.0f) - specification for floating-point input
    //                        1.0f is the initial/default value
    factorfld_ = new uiGenInput(this, tr("Multiplication Factor"),
                                FloatInpSpec(1.0f));
    
    // Position this field below the input selection dropdown
    // alignedBelow means "put this directly below that widget, left-aligned"
    // Other options: alignedAbove, rightOf, leftOf, rightAlignedBelow, etc.
    factorfld_->attach(alignedBelow, inpfld_);
    
    // -------------------------------------------------------------------------
    // CREATE SHIFT INPUT FIELD
    // -------------------------------------------------------------------------
    
    // Create another numeric input field for the shift value
    shiftfld_ = new uiGenInput(this, tr("Shift Value"),
                               FloatInpSpec(0.0f));  // Default: 0
    
    // Position this field below the factor field
    shiftfld_->attach(alignedBelow, factorfld_);
    
    // -------------------------------------------------------------------------
    // SET MAIN ALIGNMENT OBJECT
    // -------------------------------------------------------------------------
    
    // setHAlignObj() tells the dialog which widget to use for horizontal alignment
    // This determines the width of the dialog and where things line up
    // Usually set to the topmost widget
    setHAlignObj(inpfld_);
}

// =============================================================================
// DESTRUCTOR - Cleanup when dialog closes
// =============================================================================
uiMyScalerAttrib::~uiMyScalerAttrib()
{
    // Nothing to clean up for this simple dialog
    // Qt automatically deletes child widgets, so we don't need to delete
    // inpfld_, factorfld_, or shiftfld_ manually
}

// =============================================================================
// setParameters() - Load saved parameter values into UI
// Called when user opens an existing attribute for editing
// =============================================================================
bool uiMyScalerAttrib::setParameters(const Desc& desc)
{
    // -------------------------------------------------------------------------
    // VERIFY THIS IS OUR ATTRIBUTE
    // -------------------------------------------------------------------------
    
    // Make sure this description is for a MyScaler attribute
    // If user somehow opens the wrong dialog, we'll catch it here
    if (desc.attribName() != MyScaler::attribName())
        return false;  // Not our attribute, can't handle it
    
    // -------------------------------------------------------------------------
    // LOAD PARAMETER VALUES FROM DESCRIPTION
    // -------------------------------------------------------------------------
    
    // mIfGetFloat is a macro that safely retrieves a float parameter
    // Syntax: mIfGetFloat(parameter_key, variable_name, code_to_execute)
    // How it works:
    //   1. Tries to get the "factor" parameter from desc
    //   2. If found, stores value in variable named "factor"
    //   3. Executes the code: factorfld_->setValue(factor)
    //   4. If not found, does nothing (uses whatever is in the UI already)
    mIfGetFloat(MyScaler::factorStr(), factor, 
                factorfld_->setValue(factor));
    
    // Same for shift parameter
    // After this, the UI fields show the saved values
    mIfGetFloat(MyScaler::shiftStr(), shift, 
                shiftfld_->setValue(shift));
    
    // Successfully loaded parameters
    return true;
}

// =============================================================================
// setInput() - Load saved input selection into UI
// Called when user opens an existing attribute for editing
// =============================================================================
bool uiMyScalerAttrib::setInput(const Desc& desc)
{
    // putInp() is a helper method from uiAttrDescEd
    // It loads the input selection from desc into the UI widget
    // Parameters:
    //   inpfld_ - the widget to update
    //   desc    - description containing the saved input
    //   0       - input index (0 = first input)
    putInp(inpfld_, desc, 0);
    
    return true;
}

// =============================================================================
// getParameters() - Save UI values to description
// Called when user clicks OK button
// =============================================================================
bool uiMyScalerAttrib::getParameters(Desc& desc)
{
    // -------------------------------------------------------------------------
    // VERIFY THIS IS OUR ATTRIBUTE
    // -------------------------------------------------------------------------
    
    if (desc.attribName() != MyScaler::attribName())
        return false;  // Wrong attribute type
    
    // -------------------------------------------------------------------------
    // SAVE PARAMETER VALUES TO DESCRIPTION
    // -------------------------------------------------------------------------
    
    // mSetFloat is a macro that stores a float parameter
    // Syntax: mSetFloat(parameter_key, value_to_store)
    // How it works:
    //   1. Gets the value from factorfld_ using getFValue()
    //   2. Stores it in desc under the key "factor"
    //   3. OpendTect will save this value with the project
    mSetFloat(MyScaler::factorStr(), factorfld_->getFValue());
    
    // Same for shift parameter
    mSetFloat(MyScaler::shiftStr(), shiftfld_->getFValue());
    
    // Parameters saved successfully
    // If you want to validate values (e.g., factor must be > 0), do it here:
    // if (factorfld_->getFValue() <= 0)
    // {
    //     uiMSG().error(tr("Factor must be positive"));
    //     return false;  // Prevents dialog from closing
    // }
    
    return true;
}

// =============================================================================
// getInput() - Save input selection to description
// Called when user clicks OK button
// =============================================================================
bool uiMyScalerAttrib::getInput(Desc& desc)
{
    // fillInp() is a helper method from uiAttrDescEd
    // It saves the input selection from the UI widget into desc
    // Parameters:
    //   inpfld_ - the widget containing user's input selection
    //   desc    - description to update
    //   0       - input index (0 = first input)
    fillInp(inpfld_, desc, 0);
    
    return true;
}

// =============================================================================
// THAT'S IT FOR THE UI!
// Now users can configure your attribute through this dialog box
// =============================================================================
```

### 3.3 Plugin Initialization: `uimyscalerpi.cc`

Register the UI plugin:

```cpp
// =============================================================================
// UI PLUGIN INITIALIZATION FILE: uimyscalerpi.cc
// This file tells OpendTect about your UI plugin
// =============================================================================

// Include our UI class
#include "uimyscalerattrib.h"

// Include OpendTect's plugin system
#include "odplugin.h"

// =============================================================================
// UI PLUGIN INFORMATION
// Similar to the backend plugin info, but for the UI module
// =============================================================================

// Create the getODPluginInfo() function for the UI module
// Parameter "uiMyScaler" MUST match your UI module name
// NOTE: Different from backend which was "MyScaler" (no "ui" prefix)
mDefODPluginInfo(uiMyScaler)
{
    // Create PluginInfo object
    static PluginInfo retpi(
        // Display name - shown in plugin manager
        "MyScaler Plugin (GUI)",
        
        // Product name - usually "OpendTect" for free plugins
        "OpendTect",
        
        // Creator/vendor name - your name or company
        "Your Name",
        
        // Version string - should match backend version
        "1.0.0",
        
        // Description - explain this is the UI part
        "User interface for the MyScaler attribute.\n"
        "Can only be loaded into the main GUI application.");
    
    // Return pointer to the PluginInfo
    return &retpi;
}

// =============================================================================
// UI PLUGIN INITIALIZATION FUNCTION
// =============================================================================

// Create the initODPlugin() function for the UI module
// Parameter "uiMyScaler" MUST match your UI module name
mDefODInitPlugin(uiMyScaler)
{
    // IMPORTANT NOTE:
    // The UI is automatically registered by the mInitAttribUI macro
    // that we used in uimyscalerattrib.cc (line where we called mInitAttribUI)
    // That macro runs at program startup and registers the UI with the factory
    // So we don't need to manually call any initialization here!
    
    // Just return nullptr to indicate success
    return nullptr;
}

// =============================================================================
// THAT'S IT FOR THE UI MODULE!
// Summary of what happens when OpendTect starts:
//   1. OpendTect loads uiMyScaler plugin (this file)
//   2. Calls initODPlugin() (above) - does nothing, returns success
//   3. The mInitAttribUI macro (in uimyscalerattrib.cc) has already
//      registered the UI, so OpendTect knows:
//        - When user selects "MyScaler", show uiMyScalerAttrib dialog
//        - The dialog is in the "Basic" group
//   4. User can now use MyScaler attribute through the GUI!
// =============================================================================
```

### 3.4 CMakeLists.txt for UI

```cmake
# =============================================================================
# CMAKE BUILD CONFIGURATION for uiMyScaler UI Module
# This file tells CMake how to build the UI plugin
# =============================================================================

# -----------------------------------------------------------------------------
# DEPENDENCIES: What other OpendTect modules does this UI plugin need?
# -----------------------------------------------------------------------------

# This UI plugin needs TWO dependencies:
#   1. uiAttributes - provides uiAttrDescEd, uiGenInput, and other UI base classes
#   2. MyScaler - our backend module (contains the attribute logic)
# The UI depends on the backend, so it must be listed here
set(OD_MODULE_DEPS uiAttributes MyScaler)

# -----------------------------------------------------------------------------
# ORGANIZATION: Where should this plugin appear in the IDE?
# -----------------------------------------------------------------------------

# Same folder as backend for organization
# Visual Studio/Xcode will show both MyScaler and uiMyScaler in "My Plugins"
set(OD_FOLDER "My Plugins")

# -----------------------------------------------------------------------------
# PLUGIN FLAG: Tell OpendTect this is a plugin
# -----------------------------------------------------------------------------

# MUST be "yes" for plugins
set(OD_IS_PLUGIN yes)

# -----------------------------------------------------------------------------
# QT REQUIREMENT: This module uses Qt for the GUI
# -----------------------------------------------------------------------------

# OD_USEQT specifies which Qt modules we need
# "Widgets" includes all basic GUI elements (buttons, input fields, dialogs)
# This is REQUIRED for all UI modules
# Other Qt modules you might need:
#   - Core: Basic Qt functionality (usually included automatically)
#   - Network: For network communication
#   - Xml: For XML parsing
#   - Charts: For plotting charts
set(OD_USEQT Widgets)

# -----------------------------------------------------------------------------
# SOURCE FILES: What .cc files should be compiled?
# -----------------------------------------------------------------------------

# List all .cc implementation files
# Again, do NOT list .h header files
set(OD_MODULE_SOURCES
    uimyscalerattrib.cc    # UI dialog implementation
    uimyscalerpi.cc        # UI plugin initialization
)

# -----------------------------------------------------------------------------
# PLUGIN EXECUTABLES: Which OpendTect programs should load this UI plugin?
# -----------------------------------------------------------------------------

# ${OD_MAIN_EXEC} is a variable containing the main OpendTect GUI program:
#   - od_main (or OpendTect.exe on Windows)
# This UI plugin will ONLY be loaded by the main GUI program
# It will NOT be loaded by batch processing tools (they don't have a GUI)
set(OD_PLUGIN_ALO_EXEC
    ${OD_MAIN_EXEC}
)

# NOTE: This is different from the backend which used ${OD_ATTRIB_EXECS}
# The backend can be used in both GUI and batch programs
# The UI can only be used in the GUI program

# -----------------------------------------------------------------------------
# INITIALIZE MODULE: Run OpendTect's build system
# -----------------------------------------------------------------------------

# This macro:
#   - Creates the build target
#   - Links dependencies (uiAttributes and MyScaler)
#   - Links Qt libraries (because we set OD_USEQT)
#   - Sets up include paths
#   - Configures installation
OD_INIT_MODULE()

# =============================================================================
# BUILD COMPLETE!
# After building:
#   - MyScaler.so (or .dll/.dylib) - backend plugin
#   - uiMyScaler.so (or .dll/.dylib) - UI plugin
# Both must be in the plugins directory for OpendTect to find them
# =============================================================================
```

---

## Step 4: Build Configuration

### 4.1 Update Main CMakeLists.txt

Add your plugin directories to `plugins/CMakeLists.txt`:

```cmake
# =============================================================================
# REGISTERING YOUR PLUGIN WITH OPENDTECT'S BUILD SYSTEM
# =============================================================================

# Open the file: /path/to/opendtect/plugins/CMakeLists.txt
# Find the section where plugins are added (look for other OD_ADD_PLUGIN_DIR calls)
# Add these two lines:

# Add backend plugin directory
# This tells CMake to look in plugins/MyScaler/ for a CMakeLists.txt
OD_ADD_PLUGIN_DIR(MyScaler)

# Add UI plugin directory
# This tells CMake to look in plugins/uiMyScaler/ for a CMakeLists.txt
OD_ADD_PLUGIN_DIR(uiMyScaler)

# IMPORTANT: Order matters! Backend must come BEFORE UI
# Why? Because uiMyScaler depends on MyScaler
# CMake builds dependencies before things that depend on them

# After adding these lines, save the file
# Next time you run CMake, it will include your plugin in the build
```

### 4.2 Configure CMake

```bash
# =============================================================================
# CMAKE CONFIGURATION
# This step prepares the build system
# You only need to do this once (unless you change CMake files)
# =============================================================================

# Navigate to your OpendTect build directory
# (This is usually a separate directory from the source code)
cd /path/to/opendtect/build

# Run CMake to configure the build
# This analyzes your system, finds dependencies, and generates build files
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \    # Build optimized release version
    -DQT_ROOT=/path/to/qt \          # Where Qt is installed
    -DOSG_ROOT=/path/to/osg          # Where OpenSceneGraph is installed

# What each parameter means:
#
# .. (two dots) - path to source directory (parent of build directory)
#
# -DCMAKE_BUILD_TYPE=Release
#   - Release: Optimized, fast, no debug info (for production)
#   - Debug: Slower, includes debugging symbols (for development)
#   - RelWithDebInfo: Optimized but includes debug info (good compromise)
#
# -DQT_ROOT=/path/to/qt
#   - Replace with actual path to Qt installation
#   - Example (Linux): /opt/Qt/6.8.3/gcc_64
#   - Example (Windows): C:/Qt/6.8.3/msvc2022_64
#   - Example (macOS): /Users/yourname/Qt/6.8.3/clang_64
#
# -DOSG_ROOT=/path/to/osg
#   - Replace with actual path to OpenSceneGraph installation
#   - Where you built/installed OSG

# If CMake succeeds, you'll see:
#   "-- Configuring done"
#   "-- Generating done"
#   "-- Build files have been written to: /path/to/build"

# If CMake fails, read the error message carefully:
#   - Missing Qt? Check QT_ROOT path
#   - Missing OSG? Check OSG_ROOT path
#   - Missing compiler? Install build tools for your platform
```

---

## Step 5: Building and Testing

### 5.1 Build the Plugin

```bash
# =============================================================================
# BUILDING YOUR PLUGIN
# This compiles your C++ code into shared libraries (.so, .dll, or .dylib)
# =============================================================================

# Make sure you're in your build directory
cd /path/to/opendtect/build

# -----------------------------------------------------------------------------
# OPTION 1: Build only your plugins (faster, recommended during development)
# -----------------------------------------------------------------------------

cmake --build . --target MyScaler uiMyScaler

# What this does:
#   --build .           = build in current directory
#   --target MyScaler   = build only MyScaler backend plugin
#   --target uiMyScaler = build only uiMyScaler UI plugin
#
# This is MUCH faster than building all of OpendTect
# Use this when you're actively developing your plugin

# -----------------------------------------------------------------------------
# OPTION 2: Build everything (slower, but ensures dependencies are up to date)
# -----------------------------------------------------------------------------

cmake --build .

# What this does:
#   --build .  = build everything in the current directory
#
# This builds:
#   - All OpendTect modules
#   - All plugins (including yours)
#   - All executables
#
# Use this for a clean, complete build

# -----------------------------------------------------------------------------
# BUILD OUTPUT
# -----------------------------------------------------------------------------

# If build succeeds, you'll see your plugins in:
#   Linux:   build/bin/Debug/plugins/ or build/bin/Release/plugins/
#            Files: libMyScaler.so, libuiMyScaler.so
#
#   Windows: build/bin/Debug/plugins/ or build/bin/Release/plugins/
#            Files: MyScaler.dll, uiMyScaler.dll
#
#   macOS:   build/bin/Debug/plugins/ or build/bin/Release/plugins/
#            Files: libMyScaler.dylib, libuiMyScaler.dylib

# -----------------------------------------------------------------------------
# BUILD ERRORS
# -----------------------------------------------------------------------------

# If build fails:
#   1. Read the error message carefully (it's usually near the bottom)
#   2. Common errors:
#      - "No such file": Check #include statements, file paths
#      - "Undefined reference": Check that all functions are implemented
#      - "Multiple definition": Remove duplicate implementations
#   3. Fix the error and run build command again
```

### 5.2 Test the Plugin

```text
=============================================================================
TESTING YOUR PLUGIN IN OPENDTECT
Follow these steps to verify your plugin works correctly
=============================================================================

STEP 1: Launch OpendTect
-------------------------
From your build directory, run OpendTect:

Linux/macOS:
    ./bin/Debug/od_main
    (or ./bin/Release/od_main for release builds)

Windows:
    Open Visual Studio, set od_main as startup project, press F5
    (or run bin\Debug\od_main.exe from command prompt)

What to expect:
    - OpendTect main window opens
    - No error messages about MyScaler plugin
    - Check Help > About > Plugins to verify your plugin loaded

STEP 2: Load Seismic Data
--------------------------
You need seismic data to test your attribute:

Option A - Use demo data:
    - File > Open Seismic > Load Sample Data (if available)

Option B - Load your own data:
    - Survey > Select/Setup
    - Choose or create a survey with seismic data
    - File > Open Seismic > Load Cube/Volume

What to expect:
    - 3D viewer window opens
    - Seismic data is displayed
    - You can see amplitudes as colors

STEP 3: Open Attribute Dialog
------------------------------
Two ways to add attributes:

Option A - Right-click method:
    1. Right-click on the seismic display in the 3D viewer
    2. Select "Select Attributes..." from menu
    
Option B - Scene menu method:
    1. In 3D viewer, find the inline/crossline/timeslice
    2. Right-click the object name in the tree
    3. Select "Select Attributes..."

What to expect:
    - "Attribute Set" dialog opens
    - Shows current attribute (usually "Stored Cube")

STEP 4: Add Your MyScaler Attribute
------------------------------------
1. Click "Add as new" button (or "Add" if editing existing)
   - A dropdown menu appears with attribute groups

2. Expand "Basic" group
   - This is where we registered MyScaler (sKeyBasicGrp)

3. Select "MyScaler" from the list
   - Your uiMyScalerAttrib dialog opens!

What to expect:
    - Dialog titled "MyScaler" or "Attribute parameters"
    - Three fields visible:
        * Input data (dropdown)
        * Multiplication Factor (number field)
        * Shift Value (number field)

STEP 5: Configure Parameters
-----------------------------
1. Input data dropdown:
   - Should show available seismic data
   - Select the seismic cube you loaded
   - This is what will be processed

2. Multiplication Factor field:
   - Try entering: 2.0
   - This will double all amplitude values
   - You should see amplitudes become more extreme

3. Shift Value field:
   - Try entering: 0
   - This adds nothing (neutral)
   - Later try 100 to add a constant offset

What to expect:
    - All fields accept input
    - No error messages
    - OK button is enabled

STEP 6: Compute and Display
----------------------------
1. Click "OK" button
   - Dialog closes
   - Returns to Attribute Set dialog

2. Click "OK" on Attribute Set dialog
   - OpendTect starts computing your attribute
   - Progress bar may appear

3. Wait for computation to complete
   - Can take seconds to minutes depending on data size

What to expect:
    - Progress indicator shows computation
    - When done, display updates with YOUR processed data
    - Amplitudes should be 2x larger (if factor=2.0)
    
STEP 7: Verify Results
-----------------------
Check that your attribute worked:

1. Visual check:
   - If factor > 1: Colors should be more saturated/extreme
   - If factor < 1: Colors should be less saturated/subdued
   - If shift != 0: Entire color range shifts

2. Amplitude readout:
   - Hover mouse over seismic display
   - Look at amplitude value in status bar
   - Should be: original_value * factor + shift

3. Try different parameters:
   - Right-click display again > "Select Attributes..."
   - Edit MyScaler parameters
   - Try factor=0.5 (halve amplitudes)
   - Try factor=-1.0 (invert polarity)
   - Try shift=100 (add offset)

TROUBLESHOOTING
---------------
Problem: MyScaler doesn't appear in attribute list
Solution: 
    - Check Help > About > Plugins
    - Verify both MyScaler and uiMyScaler loaded
    - Check OpendTect log files for error messages
    - Rebuild plugins ensuring no build errors

Problem: Dialog opens but fields are empty/wrong
Solution:
    - Check UI code - verify field creation in constructor
    - Check setParameters() is being called
    - Add debug output to verify values

Problem: Computation fails or crashes
Solution:
    - Check computeData() for null pointer access
    - Verify getInputData() returns true
    - Check for division by zero or invalid math operations
    - Use debugger to step through computeData()

Problem: Results look wrong
Solution:
    - Print input/output values to verify math
    - Check if factor_ and shift_ have correct values
    - Verify formula: output = input * factor + shift
    - Check for undefined values (mIsUdf)
```

---

## Complete Example

Here's a more advanced example that adds an enum parameter for different operations:

### Enhanced Backend (myscalerattrib.h)

```cpp
#pragma once

#include "attribprovider.h"

namespace Attrib
{

class MyScaler : public Provider
{
public:
    static void         initClass();
                        MyScaler(Desc&);
    
    static const char*  attribName()    { return "MyScaler"; }
    static const char*  actionStr()     { return "action"; }
    static const char*  factorStr()     { return "factor"; }
    static const char*  shiftStr()      { return "shift"; }

protected:
                        ~MyScaler();
    static Provider*    createInstance(Desc&);
    static void         updateDesc(Desc&);  // NEW: dynamic parameter updates
    
    bool                allowParallelComputation() const override
                                                        { return true; }
    bool                getInputData(const BinID&, int zintv) override;
    bool                computeData(const DataHolder& output, 
                                   const BinID& relpos,
                                   int z0, int nrsamples, 
                                   int threadid) const override;
    
    int                 action_;        // NEW: 0=Scale, 1=Square, 2=Abs
    float               factor_;
    float               shift_;
    
    const DataHolder*   inputdata_;
    int                 dataidx_;
};

} // namespace Attrib
```

### Enhanced Implementation (myscalerattrib.cc)

```cpp
#include "myscalerattrib.h"
#include "attribdataholder.h"
#include "attribdesc.h"
#include "attribfactory.h"
#include "attribparam.h"
#include <math.h>

namespace Attrib
{

mAttrDefCreateInstance(MyScaler)

void MyScaler::initClass()
{
    mAttrStartInitClassWithUpdate  // Use this for dynamic updates
    
    // Add enum parameter for action type
    EnumParam* action = new EnumParam(actionStr());
    action->addEnum("Scale");
    action->addEnum("Square");
    action->addEnum("Absolute Value");
    desc->addParam(action);
    
    FloatParam* factor = new FloatParam(factorStr());
    factor->setDefaultValue(1.0f);
    factor->setLimits(Interval<float>(-1000.0f, 1000.0f));
    desc->addParam(factor);
    
    FloatParam* shift = new FloatParam(shiftStr());
    shift->setDefaultValue(0.0f);
    desc->addParam(shift);
    
    desc->addOutputDataType(Seis::UnknowData);
    desc->addInput(InputSpec("Input data", true));
    
    mAttrEndInitClass
}

void MyScaler::updateDesc(Desc& desc)
{
    // Enable/disable parameters based on action
    const BufferString action = 
        desc.getValParam(actionStr())->getStringValue();
    
    const bool isscale = action == "Scale";
    desc.setParamEnabled(factorStr(), isscale);
    desc.setParamEnabled(shiftStr(), isscale);
}

MyScaler::MyScaler(Desc& desc)
    : Provider(desc)
    , inputdata_(nullptr)
    , dataidx_(-1)
{
    if (!isOK())
        return;
    
    mGetEnum(action_, actionStr());
    mGetFloat(factor_, factorStr());
    mGetFloat(shift_, shiftStr());
}

MyScaler::~MyScaler()
{
}

bool MyScaler::getInputData(const BinID& relpos, int zintv)
{
    inputdata_ = inputs_[0]->getData(relpos, zintv);
    if (!inputdata_)
        return false;
    
    dataidx_ = getDataIndex(0);
    return true;
}

bool MyScaler::computeData(const DataHolder& output, const BinID& relpos,
                          int z0, int nrsamples, int threadid) const
{
    if (!inputdata_)
        return false;
    
    for (int idx = 0; idx < nrsamples; idx++)
    {
        const float inval = getInputValue(*inputdata_, dataidx_, idx, z0);
        float outval = inval;
        
        // Apply operation based on action
        switch (action_)
        {
            case 0: // Scale
                outval = inval * factor_ + shift_;
                break;
            case 1: // Square
                outval = inval * inval;
                break;
            case 2: // Absolute value
                outval = fabs(inval);
                break;
        }
        
        setOutputValue(output, 0, idx, z0, outval);
    }
    
    return true;
}

} // namespace Attrib
```

### Enhanced UI (uimyscalerattrib.h)

```cpp
#pragma once

#include "uiattrdesced.h"

namespace Attrib { class Desc; }
class uiAttrSel;
class uiGenInput;

class uiMyScalerAttrib : public uiAttrDescEd
{
public:
                        uiMyScalerAttrib(uiParent*, bool is2d);
                        ~uiMyScalerAttrib();

protected:
    uiAttrSel*          inpfld_;
    uiGenInput*         actionfld_;     // NEW: action selector
    uiGenInput*         factorfld_;
    uiGenInput*         shiftfld_;
    
    void                actionSel(CallBacker*);  // NEW: callback
    
    bool                setParameters(const Attrib::Desc&) override;
    bool                setInput(const Attrib::Desc&) override;
    bool                getParameters(Attrib::Desc&) override;
    bool                getInput(Attrib::Desc&) override;
    
    mDeclReqAttribUIFns
};
```

### Enhanced UI Implementation (uimyscalerattrib.cc)

```cpp
#include "uimyscalerattrib.h"
#include "myscalerattrib.h"

#include "attribdesc.h"
#include "attribparam.h"
#include "uiattrsel.h"
#include "uigeninput.h"
#include "uiattribfactory.h"

using namespace Attrib;

static const char* actionstr[] =
{
    "Scale",
    "Square",
    "Absolute Value",
    nullptr
};

mInitAttribUI(uiMyScalerAttrib, MyScaler, "MyScaler", sKeyBasicGrp())

uiMyScalerAttrib::uiMyScalerAttrib(uiParent* p, bool is2d)
    : uiAttrDescEd(p, is2d, HelpKey("myscaler", "attrib"))
{
    inpfld_ = createInpFld(is2d);
    
    // Add action selector
    actionfld_ = new uiGenInput(this, tr("Operation"),
                                StringListInpSpec(actionstr));
    mAttachCB(actionfld_->valueChanged, uiMyScalerAttrib::actionSel);
    actionfld_->attach(alignedBelow, inpfld_);
    
    factorfld_ = new uiGenInput(this, tr("Multiplication Factor"),
                                FloatInpSpec(1.0f));
    factorfld_->attach(alignedBelow, actionfld_);
    
    shiftfld_ = new uiGenInput(this, tr("Shift Value"),
                               FloatInpSpec(0.0f));
    shiftfld_->attach(alignedBelow, factorfld_);
    
    actionSel(nullptr);  // Initialize visibility
    setHAlignObj(inpfld_);
}

uiMyScalerAttrib::~uiMyScalerAttrib()
{
    detachAllNotifiers();
}

void uiMyScalerAttrib::actionSel(CallBacker*)
{
    // Show/hide parameters based on selected action
    const int action = actionfld_->getIntValue();
    const bool isscale = (action == 0);
    
    factorfld_->display(isscale);
    shiftfld_->display(isscale);
}

bool uiMyScalerAttrib::setParameters(const Desc& desc)
{
    if (desc.attribName() != MyScaler::attribName())
        return false;
    
    mIfGetEnum(MyScaler::actionStr(), action,
               actionfld_->setValue(action));
    mIfGetFloat(MyScaler::factorStr(), factor, 
                factorfld_->setValue(factor));
    mIfGetFloat(MyScaler::shiftStr(), shift, 
                shiftfld_->setValue(shift));
    
    actionSel(nullptr);
    return true;
}

bool uiMyScalerAttrib::setInput(const Desc& desc)
{
    putInp(inpfld_, desc, 0);
    return true;
}

bool uiMyScalerAttrib::getParameters(Desc& desc)
{
    if (desc.attribName() != MyScaler::attribName())
        return false;
    
    mSetEnum(MyScaler::actionStr(), actionfld_->getIntValue());
    
    if (actionfld_->getIntValue() == 0)  // Scale
    {
        mSetFloat(MyScaler::factorStr(), factorfld_->getFValue());
        mSetFloat(MyScaler::shiftStr(), shiftfld_->getFValue());
    }
    
    return true;
}

bool uiMyScalerAttrib::getInput(Desc& desc)
{
    fillInp(inpfld_, desc, 0);
    return true;
}
```

---

## Key Concepts Summary

### Backend (`Attrib::Provider`)
- **initClass()**: Register attribute with factory, define parameters
- **Constructor**: Read parameters from Desc
- **getInputData()**: Retrieve input trace data
- **computeData()**: Process trace sample by sample
- **allowParallelComputation()**: Return true for thread-safe processing

### UI (`uiAttrDescEd`)
- **Constructor**: Build UI elements
- **setParameters()**: Load parameters from Desc into UI
- **setInput()**: Load input selection into UI
- **getParameters()**: Save parameters from UI to Desc
- **getInput()**: Save input selection from UI to Desc
- **Callbacks**: Update UI when parameters change

### Important Macros
- `mAttrDefCreateInstance()`: Register attribute factory
- `mAttrStartInitClass`: Begin attribute initialization
- `mInitAttribUI()`: Register UI factory
- `mGetFloat()`, `mSetFloat()`: Get/set parameters
- `mAttachCB()`: Attach callback to UI element

---

## Tips and Best Practices

1. **Always enable parallel computation** for single trace processing (no shared state)
2. **Use clear, descriptive parameter names** (they appear in the UI)
3. **Set reasonable default values and limits** for numeric parameters
4. **Test with real data** to validate your algorithm
5. **Handle undefined values** using `mIsUdf()` macro
6. **Document your code** with comments explaining the algorithm
7. **Follow OpendTect coding style** (see programmer's manual)
8. **Use the Tutorial plugin** (`plugins/Tut/tutorialattrib.*`) as a reference

---

## Troubleshooting

### Plugin not appearing in OpendTect
- Check that CMakeLists.txt is correct
- Verify plugin builds successfully
- Ensure plugin is in correct directory (platform-specific)
- Check OpendTect logs for loading errors

### Crashes during computation
- Verify all pointers are checked before use
- Ensure array bounds are correct
- Use debugger to identify crash location

### UI not updating correctly
- Verify all callbacks are attached with `mAttachCB()`
- Check that `setParameters()` and `getParameters()` are symmetric
- Call `actionSel(nullptr)` in constructor to initialize visibility

### Build errors
- Ensure all dependencies are listed in CMakeLists.txt
- Check include paths
- Verify you're using correct OpendTect API version

---

## Further Reading

- [OpendTect Programmer's Manual](https://doc.opendtect.org/)
- [Tutorial Plugin Source Code](plugins/Tut/)
- [Other Example Plugins](plugins/)
- [OpendTect Developers Google Group](https://groups.google.com/g/opendtect-developers)

---

**Congratulations!** You now know how to create single trace processing plugins for OpendTect. Start simple, test thoroughly, and gradually add more sophisticated features.

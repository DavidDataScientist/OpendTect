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
#pragma once

#include "attribprovider.h"

namespace Attrib
{

/*!\brief MyScaler Attribute

Scales and shifts seismic trace values.

Input:
0       Input seismic data

Outputs:
0       Scaled and shifted trace
*/

class MyScaler : public Provider
{
public:
    static void         initClass();
                        MyScaler(Desc&);
    
    static const char*  attribName()    { return "MyScaler"; }
    static const char*  factorStr()     { return "factor"; }
    static const char*  shiftStr()      { return "shift"; }

protected:
                        ~MyScaler();
    static Provider*    createInstance(Desc&);
    
    // Enable parallel computation for better performance
    bool                allowParallelComputation() const override
                                                        { return true; }
    
    // Core methods - must implement these
    bool                getInputData(const BinID&, int zintv) override;
    bool                computeData(const DataHolder& output, 
                                   const BinID& relpos,
                                   int z0, int nrsamples, 
                                   int threadid) const override;
    
    // Member variables for parameters
    float               factor_;
    float               shift_;
    
    // For accessing input data
    const DataHolder*   inputdata_;
    int                 dataidx_;
};

} // namespace Attrib
```

### 2.2 Implementation File: `myscalerattrib.cc`

Implement the attribute provider:

```cpp
#include "myscalerattrib.h"
#include "attribdataholder.h"
#include "attribdesc.h"
#include "attribfactory.h"
#include "attribparam.h"

namespace Attrib
{

// Register this attribute with the factory
mAttrDefCreateInstance(MyScaler)

void MyScaler::initClass()
{
    mAttrStartInitClass
    
    // Define parameter: factor (multiplication factor)
    FloatParam* factor = new FloatParam(factorStr());
    factor->setDefaultValue(1.0f);
    factor->setLimits(Interval<float>(-1000.0f, 1000.0f));
    desc->addParam(factor);
    
    // Define parameter: shift (value to add)
    FloatParam* shift = new FloatParam(shiftStr());
    shift->setDefaultValue(0.0f);
    desc->addParam(shift);
    
    // Specify output data type
    desc->addOutputDataType(Seis::UnknowData);
    
    // Define required input
    desc->addInput(InputSpec("Input data", true));
    
    mAttrEndInitClass
}

MyScaler::MyScaler(Desc& desc)
    : Provider(desc)
    , inputdata_(nullptr)
    , dataidx_(-1)
{
    if (!isOK())
        return;
    
    // Retrieve parameter values from description
    mGetFloat(factor_, factorStr());
    mGetFloat(shift_, shiftStr());
}

MyScaler::~MyScaler()
{
}

bool MyScaler::getInputData(const BinID& relpos, int zintv)
{
    // Get input data for the current position
    inputdata_ = inputs_[0]->getData(relpos, zintv);
    if (!inputdata_)
        return false;
    
    // Get the data index for accessing trace values
    dataidx_ = getDataIndex(0);
    return true;
}

bool MyScaler::computeData(const DataHolder& output, const BinID& relpos,
                          int z0, int nrsamples, int threadid) const
{
    if (!inputdata_)
        return false;
    
    // Process each sample in the trace
    for (int idx = 0; idx < nrsamples; idx++)
    {
        // Get input value
        const float inval = getInputValue(*inputdata_, dataidx_, idx, z0);
        
        // Apply scaling and shift
        const float outval = inval * factor_ + shift_;
        
        // Set output value
        setOutputValue(output, 0, idx, z0, outval);
    }
    
    return true;
}

} // namespace Attrib
```

### 2.3 Plugin Initialization: `myscalerpi.cc`

Register the plugin with OpendTect:

```cpp
#include "myscalerattrib.h"
#include "odplugin.h"

mDefODPluginInfo(MyScaler)
{
    static PluginInfo retpi(
        "MyScaler Plugin (Base)",
        "OpendTect",
        "Your Name",
        "1.0.0",
        "Single trace scaling and shifting attribute.\n"
        "This module can be loaded into batch processing tools.");
    return &retpi;
}

mDefODInitPlugin(MyScaler)
{
    // Initialize the attribute class
    Attrib::MyScaler::initClass();
    
    return nullptr; // Return nullptr on success
}
```

### 2.4 CMakeLists.txt for Backend

```cmake
set(OD_MODULE_DEPS Attributes)
set(OD_FOLDER "My Plugins")
set(OD_IS_PLUGIN yes)

set(OD_MODULE_SOURCES
    myscalerattrib.cc
    myscalerpi.cc
)

set(OD_PLUGIN_ALO_EXEC
    ${OD_ATTRIB_EXECS}
)

OD_INIT_MODULE()
```

---

## Step 3: Create the UI Module

### 3.1 Header File: `uimyscalerattrib.h`

Create the UI class:

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
    uiAttrSel*          inpfld_;        // Input selection field
    uiGenInput*         factorfld_;     // Factor input field
    uiGenInput*         shiftfld_;      // Shift input field
    
    // Must implement these methods
    bool                setParameters(const Attrib::Desc&) override;
    bool                setInput(const Attrib::Desc&) override;
    bool                getParameters(Attrib::Desc&) override;
    bool                getInput(Attrib::Desc&) override;
    
    mDeclReqAttribUIFns
};
```

### 3.2 Implementation File: `uimyscalerattrib.cc`

Implement the UI:

```cpp
#include "uimyscalerattrib.h"
#include "myscalerattrib.h"

#include "attribdesc.h"
#include "attribparam.h"
#include "uiattrsel.h"
#include "uigeninput.h"
#include "uiattribfactory.h"

using namespace Attrib;

// Register this UI with the attribute factory
mInitAttribUI(uiMyScalerAttrib, MyScaler, "MyScaler", sKeyBasicGrp())

uiMyScalerAttrib::uiMyScalerAttrib(uiParent* p, bool is2d)
    : uiAttrDescEd(p, is2d, HelpKey("myscaler", "attrib"))
{
    // Create input selection field
    inpfld_ = createInpFld(is2d);
    
    // Create factor input field
    factorfld_ = new uiGenInput(this, tr("Multiplication Factor"),
                                FloatInpSpec(1.0f));
    factorfld_->attach(alignedBelow, inpfld_);
    
    // Create shift input field
    shiftfld_ = new uiGenInput(this, tr("Shift Value"),
                               FloatInpSpec(0.0f));
    shiftfld_->attach(alignedBelow, factorfld_);
    
    // Set the main alignment object
    setHAlignObj(inpfld_);
}

uiMyScalerAttrib::~uiMyScalerAttrib()
{
}

bool uiMyScalerAttrib::setParameters(const Desc& desc)
{
    if (desc.attribName() != MyScaler::attribName())
        return false;
    
    // Load parameter values from description
    mIfGetFloat(MyScaler::factorStr(), factor, 
                factorfld_->setValue(factor));
    mIfGetFloat(MyScaler::shiftStr(), shift, 
                shiftfld_->setValue(shift));
    
    return true;
}

bool uiMyScalerAttrib::setInput(const Desc& desc)
{
    // Load input selection
    putInp(inpfld_, desc, 0);
    return true;
}

bool uiMyScalerAttrib::getParameters(Desc& desc)
{
    if (desc.attribName() != MyScaler::attribName())
        return false;
    
    // Save parameter values to description
    mSetFloat(MyScaler::factorStr(), factorfld_->getFValue());
    mSetFloat(MyScaler::shiftStr(), shiftfld_->getFValue());
    
    return true;
}

bool uiMyScalerAttrib::getInput(Desc& desc)
{
    // Save input selection
    fillInp(inpfld_, desc, 0);
    return true;
}
```

### 3.3 Plugin Initialization: `uimyscalerpi.cc`

Register the UI plugin:

```cpp
#include "uimyscalerattrib.h"
#include "odplugin.h"

mDefODPluginInfo(uiMyScaler)
{
    static PluginInfo retpi(
        "MyScaler Plugin (GUI)",
        "OpendTect",
        "Your Name",
        "1.0.0",
        "User interface for the MyScaler attribute.\n"
        "Can only be loaded into the main GUI application.");
    return &retpi;
}

mDefODInitPlugin(uiMyScaler)
{
    // The UI is automatically registered by the mInitAttribUI macro
    // in uimyscalerattrib.cc, so we just return success here
    return nullptr;
}
```

### 3.4 CMakeLists.txt for UI

```cmake
set(OD_MODULE_DEPS uiAttributes MyScaler)
set(OD_FOLDER "My Plugins")
set(OD_IS_PLUGIN yes)
set(OD_USEQT Widgets)

set(OD_MODULE_SOURCES
    uimyscalerattrib.cc
    uimyscalerpi.cc
)

set(OD_PLUGIN_ALO_EXEC
    ${OD_MAIN_EXEC}
)

OD_INIT_MODULE()
```

---

## Step 4: Build Configuration

### 4.1 Update Main CMakeLists.txt

Add your plugin directories to `plugins/CMakeLists.txt`:

```cmake
# Add these lines to the plugin list
OD_ADD_PLUGIN_DIR(MyScaler)
OD_ADD_PLUGIN_DIR(uiMyScaler)
```

### 4.2 Configure CMake

```bash
cd /path/to/opendtect/build
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DQT_ROOT=/path/to/qt \
         -DOSG_ROOT=/path/to/osg
```

---

## Step 5: Building and Testing

### 5.1 Build the Plugin

```bash
# Build just your plugins
cmake --build . --target MyScaler uiMyScaler

# Or build everything
cmake --build .
```

### 5.2 Test the Plugin

1. **Launch OpendTect** from your build directory
2. **Load seismic data** into the viewer
3. **Open Attribute Set**:
   - Right-click on a seismic display
   - Select "Select Attributes..."
4. **Add your attribute**:
   - Click "Add as new" or "Add"
   - Choose "MyScaler" from the "Basic" group
5. **Configure parameters**:
   - Select input data
   - Set factor (e.g., 2.0 to double amplitudes)
   - Set shift (e.g., 100 to add constant offset)
6. **Compute and display** the result

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

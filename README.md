# XGrid

**XGrid: Design Exploration Tool for FPGA XGBoost**

XGrid is an open-source, end-to-end framework that converts XGBoost models into synthesizable Verilog and explores their hardware cost on FPGAs. It extends [TreeLUT](https://github.com/kiabuzz/TreeLUT.git) with an integrated input quantization module, an argmax module, and a multiplexer-based tree generator, so that the reported cost reflects a complete edge inference pipeline that starts from raw sensor data and ends with the final class.

## Overview

TreeLUT assumes that the input data is already normalized and quantized, and it leaves the argmax step to software. XGrid removes both assumptions and adds new design-exploration options.

**Changes with respect to the original TreeLUT:**
- **Input quantization module:** min-max quantization of raw inputs generated in Verilog, so the cost of the front-end is included in the results.
- **Argmax module:** the final class selection is generated in Verilog.
- **Multiplexer-based trees:** a new Verilog generator that maps each decision node to a multiplexer, avoiding the exponential number of clauses of equation-based trees.
- **Enhanced equation-based trees:** long equations are decomposed into intermediate values to reduce the load on Verilog parsers, and the code uses combinational `assign` constructions instead of behavioral Verilog.
- **Design exploration:** a grid search over the quantization widths, with and without the quantization and argmax modules.

The generated Verilog has been synthesized with Yosys, and the results reported in the paper use a generic 6-input LUT target.

## Tool Flow

1. Train an XGBoost model in Python.
2. Choose the feature width `w_feature` and the vote width `w_tree`, or a range of values to explore.
3. Choose whether to include the input quantization and argmax modules.
4. Choose the tree representation: equation-based or multiplexer-based.
5. XGrid extracts the trees and generates the Verilog code.
6. Synthesize the generated code with your FPGA toolchain.

## Installation

```bash
git clone https://github.com/Olavo-B/TreeLUT.git
cd TreeLUT
python3 -m venv env
source env/bin/activate
pip install .
```

## Getting Started

The [example](./example) Jupyter Notebook shows the full flow. At each step, a few hyperparameters can be tuned to reach the target accuracy and area.

### 1. Data Quantization

Quantize the input features into unsigned integers of `w_feature` bits, a hyperparameter of the design. Some datasets are already quantized, such as MNIST with 8-bit inputs. For raw continuous data, use min-max quantization, which preserves feature variance and ordering and outperforms direct bit truncation at low bit depths. XGrid can also generate this step in hardware, so that the circuit accepts raw inputs.

### 2. XGBoost Model Training

Train the model on the quantized data. The current version supports XGBoost classifiers.

### 3. Model Generation

The `TreeLUTClassifier` class prepares the trained model and generates the hardware files.

**Initialization parameters:**

| Parameter | Description |
|---|---|
| `xgb_model` | The XGBoost model trained in Step 2. |
| `w_feature` | Bitwidth of the quantized features. |
| `w_tree` | Quantization bitwidth of the leaf values (votes). |
| `pipeline` (optional) | Locations of the pipeline stages. See Section 4.2 of the TreeLUT paper. |
| `dir_path` (optional) | Folder in which the hardware files are generated. |

<!-- TODO: document the new options (tree representation, quantization module, argmax module, equation decomposition) with their exact names. -->

**Functions:**

| Function | Description |
|---|---|
| `convert()` | Quantizes the XGBoost model and converts it into the TreeLUT model. |
| `predict(X_test)` | Generates predictions for the given data. |
| `verilog()` | Generates the Verilog hardware files. |
| `testbench(X_test, y_test)` | Generates the Verilog testbench files. |

## Design Space Exploration

The paper explores `w_feature` and `w_tree` from 1 to 8, for two tree representations and four combinations of the optional modules, on three UCI datasets: Adult, Covertype and Dry Bean. All designs use 7 trees with a maximum depth of 7. The main findings are:

- The multiplexer-based representation reduces LUT usage compared to the equation-based one on all datasets.
- Including the quantization module increases the LUT count by about 59% to 80%, while the argmax module has a small cost.
- The best allocation between `w_feature` and `w_tree` depends on the dataset and on the area budget, so both widths should be selected jointly.

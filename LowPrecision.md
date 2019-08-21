# Low precision support in NVDLA

Use of low precision such 8-bit, 4-bit, or even lower number of bits for inference is one of the optimization methods used in deep learning. It helps to compress the model reducing memory footprint and to improve performance with a small degradation in accuracy. Using INT8 precision for inference requires quantizing pre-trained models from floating point to INT8 and programming converters in NVDLA for scaling/re-scaling tensors.

### NVDLA architecture for INT8 precision support includes the following:
-	INT8 input/output data read/write
-	32-bit internal pipeline, avoids saturation in mathematical computations
-	Per-tensor input scaling using input converters
-	Per-tensor and per-kernel output re-scaling using output converters

### Steps to generate INT8 quantized model:
-	Analyze the dynamic range of per-layer tensors and calculate scale factors using TensorRT
-	Import scale factors generated using TensorRT to NVDLA JSON format
-	Quantize model weights and determine the converter parameters using scale factors

#### Analyze dynamic range of per-layer tensors and calculate scale factors using TensorRT
A calibration tool collects the dynamic range of the output tensor for each layer over a dataset of images. This dynamic range information can be used to calculate per-tensor scale factors. For NVDLA, calibration interface TensorRT is used to generate scale factors.

Refer to https://github.com/NVIDIA/TensorRT/tree/release/5.1/samples/opensource/sampleINT8 for sample application which explains how to use TensorRT to generate scales factors.

Note: Use IInt8EntropyCalibrator2 for calibration and dump calibration scales using writeCalibrationCache() to import it in NVDLA JSON format

##### JSON schema for calibration table

The NVDLA Compiler uses the following JSON schema to import scale factors generated from TensorRT.

```
{
    "type" : "object",
    "description": "JSON schema for calibration table",
    "layer" : {
        "type": "array",
        "description": "per-layer scale factor for output tensor, scale factor can be described using either scale or min/max",
        "oneOf": ["scale", {"min", "max"}],
        "scale": {
            "type": "float",
            "description": "scale value calibrated for output tensor of layer"
        },
        "min": {
            "type": float",
            "description": "minimum value of the source precision dynamic range for output tensor of layer"
        },
        "max": {
            "type": "float",
            "description": "maximum value of the source precision dynamic range for output tensor of layer"
        },
        "offset": {
            "type" : "integer",
            "description": "offset used for asymmetric scaling, it should be 0 for symmetric scaling"
        }
    }
}
```

##### Sample calibration table for first few layers of ResNet-50 using symmetric scaling

```
{
	"data" : {
		"scale": 0.00781453,
		"min": 0,
		"max": 0,
		"offset": 0
	},
	"conv1" : {
		"scale": 0.0891214,
		"min": 0,
		"max": 0,
		"offset": 0
	},
	"pool1" : {
		"scale": 0.0891214,
		"min": 0,
		"max": 0,
		"offset": 0
	},
	"res2a_branch1" : {
		"scale": 0.119546,
		"min": 0,
		"max": 0,
		"offset": 0
	}
}
```

#### Quantize model weights and determine the converter parameters

The NVDLA Compiler has the ability to quantize model weights and determine the converter parameters using the scale factors from the calibration table.
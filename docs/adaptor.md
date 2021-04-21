Adaptor
=======

## Introduction

Intel® Low Precision Optimization Tool built the low-precision inference
solution on popular Deep Learning frameworks such as TensorFlow, PyTorch,
MXNet, and ONNX Runtime. The adaptor layer is the bridge between the LPOT
tuning strategy and framework vanilla quantization APIs.

## Adaptor Design

Intel® Low Precision Optimization Tool supports new adaptor extension by
implementing a subclass `Adaptor` class in the lpot.adaptor package
and registering this strategy by the `adaptor_registry` decorator.

For example, a user can implement an `Abc` adaptor like below:
```python
@adaptor_registry
class AbcAdaptor(Adaptor):
    def __init__(self, framework_specific_info):
        ...

    def quantize(self, tune_cfg, model, dataloader, q_func=None):
        ...

    def evaluate(self, model, dataloader, postprocess=None,
                 metric=None, measurer=None, iteration=-1, tensorboard=False):
        ...

    def query_fw_capability(self, model):
        ...

    def query_fused_patterns(self, model):
        ...
```

`quantize` function is used to do calibration and quantization in post-training quantization.
`evaluate` function is used to run evaluation on a validation dataset.
`query_fw_capability` function is used to run query framework quantization capability and intersects with the user yaml configuration setting to
`query_fused_patterns` function is used to run query framework graph fusion capability and decide the fusion tuning space.

### Query API

#### **Background**
Besides the adaptor API, we also introduced the Query API which describes the
behavior of the specific framework. With this API, LPOT can easily query the
following information on the current runtime framework.
*  The runtime version information;
*  The Quantizable ops' type;
*  The supported sequence of each quantizable op;
*  The instance of each sequence.

In the past, the above information was generally defined and hidden in every corner of the code which made effective maintenance difficult. With the Query API, we only need to create one unified yaml file and call the corresponding API to get the information. For example, the [tensorflow.yaml](../lpot/adaptor/tensorflow.yaml) keeps the current Tensorflow framework ability. We recommend that the end user not make modifications if requirements are not clear.

#### **Unify Config Introduction**
Below is a fragment of the Tensorflow configuration file.

* **precisions** field defines the supported precision for LPOT.
    -  valid_mixed_precision enumerates all supported precision combinations for specific scenario. For example, if one hardware doesn't support bf16， it should be `int8 + fp32`.
* **ops** field defines the valid OP type list for each precision.
* **capabilities** field focuses on the quantization ability of specific ops such as granularity, scheme, and algorithm. Note that the activation here means the input activation of the op rather than its output.
* **patterns** field defines the supported fusion sequence of each op.

```yaml
---
-
  version:
    name: '2.4.0'
  
  precisions: &common_precisions
    names: int8, uint8, bf16, fp32
    valid_mixed_precisions: []
  
  ops: &common_ops
    int8: ['Conv2D', 'MatMul', 'ConcatV2', 'MaxPool', 'AvgPool']
    uint8: ['Conv2D', 'DepthwiseConv2dNative', 'MatMul', 'ConcatV2', 'MaxPool', 'AvgPool']
    bf16: ['Conv2D']  #TODO need to add more bf16 op types here
    fp32: ['*'] # '*' means all op types
  
  capabilities: &common_capabilities
    int8: &ref_2_4_int8 {
          'Conv2D': {
            'weight': {
                        'dtype': ['int8', 'fp32'],
                        'scheme': ['sym'],
                        'granularity': ['per_channel','per_tensor'],
                        'algorithm': ['minmax']
                        },
            'activation': {
                        'dtype': ['int8', 'fp32'],
                        'scheme': ['sym'],
                        'granularity': ['per_tensor'],
                        'algorithm': ['minmax', 'kl']
                        }
                    },
          'MatMul': {
            'weight': {
                        'dtype': ['int8', 'fp32'],
                        'scheme': ['sym'],
                        'granularity': ['per_tensor'],
                        'algorithm': ['minmax', 'kl']
                        },
            'activation': {
                        'dtype': ['int8', 'fp32'],
                        'scheme': ['asym', 'sym'],
                        'granularity': ['per_tensor'],
                        'algorithm': ['minmax']
                        }
                    },
          'default': {
            'activation': {
                        'dtype': ['uint8', 'fp32'],
                        'algorithm': ['minmax'],
                        'scheme': ['sym'],
                        'granularity': ['per_tensor']
                        }
                    },
          }

    uint8: &ref_2_4_uint8 {
          'Conv2D': {
            'weight': {
                        'dtype': ['int8', 'fp32'],
                        'scheme': ['sym'],
                        'granularity': ['per_channel','per_tensor'],
                        'algorithm': ['minmax']
                        },
            'activation': {
                        'dtype': ['uint8', 'fp32'],
                        'scheme': ['sym'],
                        'granularity': ['per_tensor'],
                        'algorithm': ['minmax', 'kl']
                        }
                    },
          'MatMul': {
            'weight': {
                        'dtype': ['int8', 'fp32'],
                        'scheme': ['sym'],
                        'granularity': ['per_tensor'],
                        'algorithm': ['minmax', 'kl']
                        },
            'activation': {
                        'dtype': ['uint8', 'fp32'],
                        'scheme': ['asym', 'sym'],
                        'granularity': ['per_tensor'],
                        'algorithm': ['minmax']
                        }
                    },
          'default': {
            'activation': {
                        'dtype': ['uint8', 'fp32'],
                        'algorithm': ['minmax'],
                        'scheme': ['sym'],
                        'granularity': ['per_tensor']
                        }
                    },
          }

  patterns: &common_patterns
    fp32: [ #TODO Add more patterns here to demonstrate our concept the results external engine should return.
        'Conv2D + Add + Relu',
        'Conv2D + Add + Relu6',
        'Conv2D + Relu',
        'Conv2D + Relu6',
        'Conv2D + BiasAdd'
        ]
    int8: ['Conv2D + BiasAdd', 'Conv2D + BiasAdd + Relu', 'Conv2D + BiasAdd + Relu6']
    uint8: [
        'Conv2D + BiasAdd + AddN + Relu',
        'Conv2D + BiasAdd + AddN + Relu6',
        'Conv2D + BiasAdd + AddV2 + Relu',
        'Conv2D + BiasAdd + AddV2 + Relu6',
        'Conv2D + BiasAdd + Add + Relu',
        'Conv2D + BiasAdd + Add + Relu6',
        'Conv2D + BiasAdd + Relu',
        'Conv2D + BiasAdd + Relu6',
        'Conv2D + Add + Relu',
        'Conv2D + Add + Relu6',
        'Conv2D + Relu',
        'Conv2D + Relu6',
        'Conv2D + BiasAdd',
        'DepthwiseConv2dNative + BiasAdd + Relu6',
        'DepthwiseConv2dNative + Add + Relu6',
        'DepthwiseConv2dNative + BiasAdd',
        'MatMul + BiasAdd + Relu',
        'MatMul + BiasAdd',
  ]
```
#### **Query API Introduction**
The abstract class `QueryBackendCapability` is defined in [query.py](../lpot/adaptor/query.py#L21). Each framework should inherit it and implement the member function if needed. Refer to Tensorflow implementation [TensorflowQuery](../lpot/adaptor/tensorflow.py#L628)


## Customize a New Framework Backend

Let us take onnxruntime as an example. ONNX Runtime is a backend proposed by Microsoft, and it's based on the MLAS kernel by default.
Onnxruntime already has  [quantization tools](https://github.com/microsoft/onnxruntime/tree/master/onnxruntime/python/tools/quantization), so the question becomes how to integrate onnxruntime quantization tools into LPOT.

1. Capability
   
   The user should explore quantization capability at first. According to [onnx_quantizer](https://github.com/microsoft/onnxruntime/blob/503b61d897074a494f5798069308ee67d8fb9ace/onnxruntime/python/tools/quantization/onnx_quantizer.py#L77), the quantization tools support the following attributes:
   1.1 whether per_channel
   1.2 whether reduce_range
   1.3 QLinear mode or Integer mode (which is only seen in onnxruntime)
   1.4 whether static (static quantization or dynamic quantization)
   1.4 weight_qtype (choices are float32, int8 and uint8)
   1.5 input_qtype (choices are float32, int8 and uint8)
   1.6 quantization_params (None if dynamic quantization)
   1.7 &1.8 nodes_to_quantize, nodes_to_exclude
   1.9 op_types_to_quantize

   so we can pass a tune capability to LPOT such as:

   ```yaml
   {'optypewise': {'conv': 
                   {
                    'activation': { 'dtype': ['uint8', 'fp32']},
                    'weight': {'dtype': ['int8', 'fp32']},
                    'algorithm': ['minmax', ],
                    'granularity': ['per_channel']
                   }, 
                   'matmul': 
                   {
                    'activation': { 'dtype': ['uint8', 'fp32']},
                    'weight': {'dtype': ['int8', 'fp32']},
                    'algorithm': ['minmax', ],
                    'granularity': ['per_channel']
                   }
                   }, 
    'opwise':  {('conv1', 'conv'):
                   {
                    'activation': { 'dtype': ['uint8', 'fp32']},
                    'weight': {'dtype': ['int8', 'fp32']}
                   }
                   }
    }
   ```

2. Parse tune config
   
   LPOT will generate a tune config from your tune capability such as the
   following: 

   ```yaml
    {
        'fuse': {'int8': [['CONV2D', 'RELU', 'BN'], ['CONV2D', 'RELU']],
        'fp32': [['CONV2D', 'RELU', 'BN']]}, 
        'calib_iteration': 10,
        'op': {
        ['op1', 'CONV2D']: {
            'activation':  {'dtype': 'uint8',
                            'algorithm': 'minmax',
                            'scheme':'sym',
                            'granularity': 'per_tensor'},
            'weight': {'dtype': 'int8',
                        'algorithm': 'kl',
                        'scheme':'asym',
                        'granularity': 'per_channel'}
        },
        ['op2', 'RELU]: {
            'activation': {'dtype': 'int8',
                            'scheme': 'asym',
                            'granularity': 'per_tensor',
                            'algorithm': 'minmax'}
        },
        ['op3', 'CONV2D']: {
            'activation':  {'dtype': 'fp32'},
            'weight': {'dtype': 'fp32'}
        },
        ...
        }
    }
   ```
   Then you can parse this config into a format that ONNXQuantizer can accept.
   Verify whether your quantization API supports model wise or op wise quantization. For example, node "conv1" uses the "minmax" algorithm and node "conv2" uses the "KL" algorithm, or the whole model must use "minmax" or "KL" in general.

3. Pre-optimize
   If your backend supports FP32 graph optimization, you can apply it in **query_fw_capability** and quantize your optimized fp32 model instead of
   the original model
   >model = self.pre_optimized_model if self.pre_optimized_model else model

4. Do quantization
   
   This part depends on your backend implementations. Refer to [onnxruntime](../lpot/adaptor/onnxrt.py) as an example.

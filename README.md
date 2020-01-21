# omnigan
- [omnigan](#omnigan)
  - [Current Model](#current-model)
    - [Summary](#summary)
    - [Generator](#generator)
  - [interfaces](#interfaces)
    - [batches](#batches)
    - [data](#data)
    - [losses](#losses)
  - [Logging on comet](#logging-on-comet)
    - [Parameters](#parameters)
    - [Tests](#tests)
  - [Resources](#resources)
  - [Model Architecture](#model-architecture)
    - [Generator](#generator-1)

## Current Model

### Summary

Summary from `torchsummary` with only 1 ResBlock in the encoder and 1 in the decoders:

```
================================================================
Total params: 9,766,807
Trainable params: 9,766,807
Non-trainable params: 0
----------------------------------------------------------------
Input size (MB): 0.75
Forward/backward pass size (MB): 1746.82
Params size (MB): 37.26
Estimated Total Size (MB): 1784.82
----------------------------------------------------------------
```

Set `test_summary` to `True` in `tests/test_gen.py` to view the full summary.

**n.b.**: the adaptation decoder is not taken into account in the summary as its computations are not used in `OmniGenerator.forward(...)` and only one translation decoder is used so numbers above are a lower bound.

### Generator

High-level model in `generator.py`, building-blocks in `blocks.py`

* **Encoder**: Resnet-based Content Encoder from MUNIT
  * image => 64 (=`encoder.dim`) channels with 1 conv layer, same size
  * conv-based downsamplings (`encoder.n_downsample` times)
  * resblocks (`encoder.n_res` blocks)
* **Decoders**: Resnet-based Decoders from MUNIT for all tasks but the translation
  * resblocks projections (`decoder.n_res` blocks)
  * Sequence of `nn.Upsampling > Conv2dBlock` (`decoder.n_upsample` times)
    * should match `encoder.n_downsample`
  * final conv to get a feature map with 1 (`h`, `d`, `w`), 3 (`a`) or 19 (`s`) channels
* **Translation decoder**: SPADEResnet-based Decoder inspired by MUNIT and SPADE
  * Conditioning the translation by `SPADE([h, d, s, w])`

## interfaces

### batches
```python
batch = Dict({
    "data": {
        "d": depthmap,
        "h": heightmap,
        "w": water_segmentation_map,
        "s": segmentation_map,
        "x": real_flooded_image,
    },
    "paths":{
        "d": depthmap_path,
        "h": heightmap_path,
        "w": water_segmentation_map_path,
        "s": segmentation_map_path,
        "x": real_flooded_image_path,
    }
    "domain": rf | rn | sf | sn,
    "mode": train | val
})
```

### data

```yaml
# data file ; one for each r|s-f|n
- x: /path/to/image
  h: /path/to/height map
  d: /path/to/depth map
  w: /path/to/water map
  s: /path/to/segmentation map
- x: /path/to/another image
  d: /path/to/depth map
  s: /path/to/segmentation map
- x: ...
```

or

```json
[
    {
        "x": "/Users/victor/Documents/ccai/github/omnigan/example_data/gsv_000005.jpg",
        "s": "/Users/victor/Documents/ccai/github/omnigan/example_data/gsv_000005.npy",
        "d": "/Users/victor/Documents/ccai/github/omnigan/example_data/gsv_000005_depth.jpg"
    },
    {
        "x": "/Users/victor/Documents/ccai/github/omnigan/example_data/gsv_000006.jpg",
        "s": "/Users/victor/Documents/ccai/github/omnigan/example_data/gsv_000006.npy",
        "d": "/Users/victor/Documents/ccai/github/omnigan/example_data/gsv_000006_depth.jpg"
    }
]
```

loaders

```
loaders = Dict({
    train: { rn: loader, rf: loader, sn: loader, sf: loader},
    val: { rn: loader, rf: loader, sn: loader, sf: loader}
})
```

### losses

`trainer.losses` is a dictionary mapping to loss functions to optimize for the 3 main parts of the architecture: generator `G`, discriminators `D`, domain classifier `C`:

```python
trainer.losses = {
    "G":{ # generator
        "gan": { # gan loss from the discriminators
            "a": func, # adaptation decoder
            "t": func # translation decoder
        },
        "cycle": { # cycle-consistency loss
            "a": func,
            "t": func
        },
        "auto": { # auto-encoding loss a.k.a. reconstruction loss
            "a": l1 | l2,
            "t": l1 | l2
        },
        "tasks": {  # specific losses for each auxillary task
            "d": func, # depth estimation
            "h": func, # height estimation
            "s": cross_entropy_2d, # segmentation
            "w": func, # water generation
        },
        "classifier": l1 | l2 | CE # loss from fooling the classifier
    },
    "D":{}, # discriminator losses from the generator and true data
    "C": l1 | l2 | CE # classifier should predict the right 1-h vector [rf, rn, sf, sn]
}
```

## Logging on comet

Comet.ml will look for api keys in the following order: argument to the `Experiment(api_key=...)` call, `COMET_API_KEY` environment variable, `.comet.config` file in the current working directory, `.comet.config` in the current user's home directory.

If your not managing several comet accounts at the same time, I recommend putting `.comet.config` in your home as such:

```
[comet]
api_key=<api_key>
workspace=vict0rsch
rest_api_key=<rest_api_key>
```

### Parameters

Set `train.log_level` in your configuration file to control the amount of logging on comet:

* `0`: no logging on comet
* `1`: only aggregated losses (representational loss, translation loss, total loss)
* `2`: all losses (aggregated + task losses + auto-encoding losses)

### Tests

There's a `test_comet.py` test which will automatically start and stop an experiment, check that logging works and so on. Not to pollute your workspace, such functional tests are deleted when the test is passed through Comet's REST API which is why you need to specify this `rest_api_key` field.

Set `should_delete` to False in the file not to delete the test experiment once it has ended. You'll be able to find all your test experiments which were not deleted using the `is_functional_test` parameter on Comet's web interface.

## Resources

[Tricks and Tips for Training a GAN](https://chloes-dl.com/2019/11/19/tricks-and-tips-for-training-a-gan/)
[GAN Hacks](https://github.com/soumith/ganhacks)

## Model Architecture


### Generator

1 Resblock in Encoder, 1 in each Decoder

```
OmniGenerator(
  (encoder): Encoder(

        (model): Sequential(
        (0): Conv2dBlock(
            (pad): ReflectionPad2d((3, 3, 3, 3))
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(3, 64, kernel_size=(7, 7), stride=(1, 1))
        )
        (1): Conv2dBlock(
            (pad): ReflectionPad2d((1, 1, 1, 1))
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(64, 128, kernel_size=(4, 4), stride=(2, 2))
        )
        (2): Conv2dBlock(
            (pad): ReflectionPad2d((1, 1, 1, 1))
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(128, 256, kernel_size=(4, 4), stride=(2, 2))
        )
        (3): ResBlocks(
            (model): Sequential(
            (0): ResBlock(
                (model): Sequential(
                (0): Conv2dBlock(
                    (pad): ReflectionPad2d((1, 1, 1, 1))
                    (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                    (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                )
                (1): Conv2dBlock(
                    (pad): ReflectionPad2d((1, 1, 1, 1))
                    (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                )
                )
            )
            )
        )
        )

  )
  (decoders): ModuleDict(

    (a): ModuleDict(

        (r): AdaptationDecoder(

            (model): Sequential(
            (0): ResBlocks(
                (model): Sequential(
                (0): ResBlock(
                    (model): Sequential(
                    (0): Conv2dBlock(
                        (pad): ReflectionPad2d((1, 1, 1, 1))
                        (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                        (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                        (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                    (1): Conv2dBlock(
                        (pad): ReflectionPad2d((1, 1, 1, 1))
                        (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                        (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                    )
                )
                )
            )
            (1): Upsample(scale_factor=2.0, mode=nearest)
            (2): Conv2dBlock(
                (pad): ReflectionPad2d((2, 2, 2, 2))
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(256, 128, kernel_size=(5, 5), stride=(1, 1))
            )
            (3): Upsample(scale_factor=2.0, mode=nearest)
            (4): Conv2dBlock(
                (pad): ReflectionPad2d((2, 2, 2, 2))
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(128, 64, kernel_size=(5, 5), stride=(1, 1))
            )
            (5): Conv2dBlock(
                (pad): ReflectionPad2d((3, 3, 3, 3))
                (activation): Tanh()
                (conv): Conv2d(64, 3, kernel_size=(7, 7), stride=(1, 1))
            )
            )

      )

      (s): AdaptationDecoder(

            (model): Sequential(
            (0): ResBlocks(
                (model): Sequential(
                (0): ResBlock(
                    (model): Sequential(
                    (0): Conv2dBlock(
                        (pad): ReflectionPad2d((1, 1, 1, 1))
                        (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                        (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                        (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                    (1): Conv2dBlock(
                        (pad): ReflectionPad2d((1, 1, 1, 1))
                        (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                        (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                    )
                )
                )
            )
            (1): Upsample(scale_factor=2.0, mode=nearest)
            (2): Conv2dBlock(
                (pad): ReflectionPad2d((2, 2, 2, 2))
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(256, 128, kernel_size=(5, 5), stride=(1, 1))
            )
            (3): Upsample(scale_factor=2.0, mode=nearest)
            (4): Conv2dBlock(
                (pad): ReflectionPad2d((2, 2, 2, 2))
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(128, 64, kernel_size=(5, 5), stride=(1, 1))
            )
            (5): Conv2dBlock(
                (pad): ReflectionPad2d((3, 3, 3, 3))
                (activation): Tanh()
                (conv): Conv2d(64, 3, kernel_size=(7, 7), stride=(1, 1))
            )
            )
        )

    )

    (d): DepthDecoder(

        (model): Sequential(
            (0): ResBlocks(
            (model): Sequential(
                (0): ResBlock(
                (model): Sequential(
                    (0): Conv2dBlock(
                    (pad): ReflectionPad2d((1, 1, 1, 1))
                    (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                    (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                    (1): Conv2dBlock(
                    (pad): ReflectionPad2d((1, 1, 1, 1))
                    (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                )
                )
            )
            )
            (1): Upsample(scale_factor=2.0, mode=nearest)
            (2): Conv2dBlock(
            (pad): ReflectionPad2d((2, 2, 2, 2))
            (norm): LayerNorm()
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(256, 128, kernel_size=(5, 5), stride=(1, 1))
            )
            (3): Upsample(scale_factor=2.0, mode=nearest)
            (4): Conv2dBlock(
            (pad): ReflectionPad2d((2, 2, 2, 2))
            (norm): LayerNorm()
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(128, 64, kernel_size=(5, 5), stride=(1, 1))
            )
            (5): Conv2dBlock(
            (pad): ReflectionPad2d((3, 3, 3, 3))
            (activation): Tanh()
            (conv): Conv2d(64, 1, kernel_size=(7, 7), stride=(1, 1))
            )
        )

    )

    (s): SegmentationDecoder(

        (model): Sequential(
            (0): ResBlocks(
            (model): Sequential(
                (0): ResBlock(
                (model): Sequential(
                    (0): Conv2dBlock(
                    (pad): ReflectionPad2d((1, 1, 1, 1))
                    (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                    (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                    (1): Conv2dBlock(
                    (pad): ReflectionPad2d((1, 1, 1, 1))
                    (norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (conv): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1))
                    )
                )
                )
            )
            )
            (1): Upsample(scale_factor=2.0, mode=nearest)
            (2): Conv2dBlock(
            (pad): ReflectionPad2d((2, 2, 2, 2))
            (norm): LayerNorm()
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(256, 128, kernel_size=(5, 5), stride=(1, 1))
            )
            (3): Upsample(scale_factor=2.0, mode=nearest)
            (4): Conv2dBlock(
            (pad): ReflectionPad2d((2, 2, 2, 2))
            (norm): LayerNorm()
            (activation): LeakyReLU(negative_slope=0.2, inplace=True)
            (conv): Conv2d(128, 64, kernel_size=(5, 5), stride=(1, 1))
            )
            (5): Conv2dBlock(
            (pad): ReflectionPad2d((3, 3, 3, 3))
            (activation): Tanh()
            (conv): Conv2d(64, 19, kernel_size=(7, 7), stride=(1, 1))
            )
        )

    )

    (t): ModuleDict(

      (f): TranslationDecoder(

            (model): Sequential(
            (0): SpadeResBlocks(
                (model): Sequential(
                (0): SPADEResnetBlock(
                    (conv_0): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (conv_1): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (norm_0): SPADE(
                    (param_free_norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (mlp_shared): Sequential(
                        (0): Conv2d(24, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                        (1): ReLU()
                    )
                    (mlp_gamma): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (mlp_beta): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    )
                    (norm_1): SPADE(
                    (param_free_norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (mlp_shared): Sequential(
                        (0): Conv2d(24, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                        (1): ReLU()
                    )
                    (mlp_gamma): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (mlp_beta): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    )
                )
                )
            )
            (1): Upsample(scale_factor=2.0, mode=nearest)
            (2): Conv2dBlock(
                (pad): ZeroPad2d(padding=(2, 2, 2, 2), value=0.0)
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(256, 128, kernel_size=(5, 5), stride=(1, 1))
            )
            (3): Upsample(scale_factor=2.0, mode=nearest)
            (4): Conv2dBlock(
                (pad): ZeroPad2d(padding=(2, 2, 2, 2), value=0.0)
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(128, 64, kernel_size=(5, 5), stride=(1, 1))
            )
            (5): Conv2dBlock(
                (pad): ReflectionPad2d((3, 3, 3, 3))
                (activation): Tanh()
                (conv): Conv2d(64, 3, kernel_size=(7, 7), stride=(1, 1))
            )
            )

      )

      (n): TranslationDecoder(

            (model): Sequential(
            (0): SpadeResBlocks(
                (model): Sequential(
                (0): SPADEResnetBlock(
                    (conv_0): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (conv_1): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (norm_0): SPADE(
                    (param_free_norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (mlp_shared): Sequential(
                        (0): Conv2d(24, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                        (1): ReLU()
                    )
                    (mlp_gamma): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (mlp_beta): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    )
                    (norm_1): SPADE(
                    (param_free_norm): InstanceNorm2d(256, eps=1e-05, momentum=0.1, affine=False, track_running_stats=False)
                    (mlp_shared): Sequential(
                        (0): Conv2d(24, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                        (1): ReLU()
                    )
                    (mlp_gamma): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    (mlp_beta): Conv2d(128, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
                    )
                )
                )
            )
            (1): Upsample(scale_factor=2.0, mode=nearest)
            (2): Conv2dBlock(
                (pad): ZeroPad2d(padding=(2, 2, 2, 2), value=0.0)
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(256, 128, kernel_size=(5, 5), stride=(1, 1))
            )
            (3): Upsample(scale_factor=2.0, mode=nearest)
            (4): Conv2dBlock(
                (pad): ZeroPad2d(padding=(2, 2, 2, 2), value=0.0)
                (norm): LayerNorm()
                (activation): LeakyReLU(negative_slope=0.2, inplace=True)
                (conv): Conv2d(128, 64, kernel_size=(5, 5), stride=(1, 1))
            )
            (5): Conv2dBlock(
                (pad): ReflectionPad2d((3, 3, 3, 3))
                (activation): Tanh()
                (conv): Conv2d(64, 3, kernel_size=(7, 7), stride=(1, 1))
            )
            )

      )

    )

  )

)
```

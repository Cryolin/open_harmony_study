[TOC]

# audio学习

## audio_framework部件

### 编译构建

#### bundle.json

audio_framework是**部件名称**，该部件名称配置在其bundle.json文件中：

```json
{
    "name": "@ohos/audio_framework",
    "description": "Audio standard provides managers and provides the audio resources to application for play/record audio",
    "version": "4.0",
    "license": "Apache License 2.0",
    "publishAs": "code-segment",
    "segment": {
        "destPath": "foundation/multimedia/audio_framework"
    },
    "dirs": {},
    "scripts": {},
    "component": {
        "name": "audio_framework",
        "subsystem": "multimedia",
        "syscap": [
          "SystemCapability.Multimedia.Audio.Core",
          "SystemCapability.Multimedia.Audio.Renderer",
		  ...
        ],
        "features": [
          "audio_framework_feature_dtmf_tone",
          "audio_framework_feature_opensl_es"
        ],
        "adapted_system_type": [ "standard" ],
        "rom": "4500KB",
        "ram": "11000KB",
        "hisysevent_config": [ "//foundation/multimedia/audio_framework/hisysevent.yaml" ],
        "deps": {
          "components": [
            "ability_base",
            "ability_runtime",
            ...
            ],
          "third_party": [
            "bounds_checking_function",
            "glib",
            "libsnd",
            "libxml2",
            "pulseaudio"
          ]
        },
        ... // 略
    }
}

```

[^说明]: 还有个部件：audio_standard，与本部件功能类似，该部件已不维护。

依赖三方库pulseaudio，其部件定义文件如下：

```json
{
    "name": "@ohos/pulseaudio",
	... // 略
    "component": {
        "name": "pulseaudio",
        "subsystem": "thirdparty",
        ...	// 略
    }
}
```

**[multimedia_audio_standard](https://gitee.com/openharmony/multimedia_audio_standard)**仓的目录结构：

```
/foundation/multimedia/audio_framework  # 音频组件业务代码
├── frameworks                         # 框架代码
│   ├── native                         # 内部接口实现
│   └── js                             # 外部接口实现
│       └── napi                       # napi 外部接口实现
├── interfaces                         # 接口代码
│   ├── inner_api                      # 内部接口
│   └── kits                           # 外部接口
├── sa_profile                         # 服务配置文件
├── services                           # 服务代码
├── LICENSE                            # 证书文件
└── ohos.build                         # 编译文件
```

[^部件与模块]: multimedia_audio_framework是一个部件，每个部件有自己的build.json配置文件，每个部件中，有很多模块，每个模块有自己的BUILD.gn文件。

下面展示了multimedia_audio_framework部件bundle.json的剩余内容：

```json
{
    "name": "@ohos/audio_framework",
    ... // 略
    "component": {
        "name": "audio_framework",
        ... // 略
        "build": {
          "group_type": {
            "base_group": [
            ],
            "fwk_group": [
              "//foundation/multimedia/audio_framework/frameworks/js/napi:audio",
              "//foundation/multimedia/audio_framework/frameworks/native/ohaudio:ohaudio",
              "//foundation/multimedia/audio_framework/frameworks/native/opensles:opensles",
              "//foundation/multimedia/audio_framework/frameworks/native/audiocompatibility:audio_renderer_gateway",
              "//foundation/multimedia/audio_framework/frameworks/native/audiocompatibility:audio_capturer_gateway"
            ],
            "service_group": [
              "//foundation/multimedia/audio_framework/sa_profile:audio_service_sa_profile",
              "//foundation/multimedia/audio_framework/services/audio_service:audio_service_packages",
              "//foundation/multimedia/audio_framework/sa_profile:audio_policy_service_sa_profile",
              "//foundation/multimedia/audio_framework/services/audio_policy:audio_policy_packages",
              "//third_party/pulseaudio/ohosbuild:pulseaudio_packages",
              "//foundation/multimedia/audio_framework/frameworks/native/pulseaudio/modules:pa_extend_modules"
            ]
          },
          "inner_kits": [
            {
              "type": "none",
              "name": "//foundation/multimedia/audio_framework/services/audio_service:audio_client",
              "header": {
                "header_files": [
                  "audio_system_manager.h",
                  "audio_stream_manager.h",
                  "audio_info.h"
                ],
                "header_base": [
                    "//foundation/multimedia/audio_framework/interfaces/inner_api/native/audiomanager/include",
                    "//foundation/multimedia/audio_framework/interfaces/inner_api/native/audiocommon/include"
                ]
              }
            },
          ],
          "test": [
            "//foundation/multimedia/audio_framework/frameworks/native/audiorenderer:audio_renderer_test_packages",
			... // 略
          ]
        }
    }
}
```

[^新增模块]: 新增模块都需先在对应模块目录创建BUILD.gn文件，然后根据在部件的bundle.json通过"group_type"、"sub_component"、"inner_kits"或"test"完成与部件的关联

#### BUILD.gn

BUILD.gn中配置了模块信息，根据这些模块在部件的bundle.json的关联方式，分别讨论。

##### 通过group_type关联到部件

举例来说，audio_framework依赖的模块可以是动态库（ohos_shared_library），也可能是一个模块组合（group）

```json
"group_type": {
    "base_group": [
    ],
    "fwk_group": [
        // 动态库
        "//foundation/multimedia/audio_framework/frameworks/js/napi:audio",
    ],
    "service_group": [
        // group
        "//foundation/multimedia/audio_framework/services/audio_service:audio_service_packages",
    ]
},
```

**动态库：**

napi:audio模块的BUILD.gn文件如下

```json
import("//build/ohos.gni")
import("//build/ohos/ace/ace.gni")
import("../../../config.gni")
import("../../../multimedia_aafwk.gni")

// 动态库audio
ohos_shared_library("audio") {
  // Sanitizer配置，每项都是可选的，默认为false/空
  sanitize = {
    cfi = true  // 控制流完整性检测 : 打开
    cfi_cross_dso = true  // 跨so调用的控制流完整性检测 : 打开
    debug = false  // 调测模式 : 关闭
    blocklist = "../../../cfi_blocklist.txt"  // 屏蔽名单路径
  }
  include_dirs = [
    "audio_common/include",
    "audio_renderer/include",
    "audio_capturer/include",
    "audio_manager/include",
    "audio_stream_manager/include",
    "../../../interfaces/kits/js/audio_manager/include",
    "../../../interfaces/kits/js/audio_capturer/include",
    "../../../interfaces/kits/js/audio_renderer/include",
    "../../../interfaces/inner_api/native/audiocommon/include",
    "//third_party/libuv/include",
  ]

  // 包含的C或C++文件，如：["",""]
  sources = [
    "audio_capturer/src/audio_capturer_callback_napi.cpp",
    "audio_capturer/src/audio_capturer_device_change_callback_napi.cpp",
  ]
  // 部件内模块依赖
  deps = [
    "../../../services/audio_policy:audio_policy_client",
    "../../../services/audio_service:audio_client",
    "../../native/audiocapturer:audio_capturer",
    "../../native/audiorenderer:audio_renderer",
  ]

  defines = []
  if (audio_framework_feature_dtmf_tone) {
    defines += [ "FEATURE_DTMF_TONE" ]

    include_dirs += [
      "../../../interfaces/kits/js/toneplayer/include",
      "../../../interfaces/inner_api/native/toneplayer/include",
    ]

    sources += [ "toneplayer/src/toneplayer_napi.cpp" ]

    deps += [ "../../native/toneplayer:audio_toneplayer" ]
  }
  // 外部模块依赖
  external_deps = [
    "ability_runtime:abilitykit_native",
    "ability_runtime:napi_base_context",
    "c_utils:utils",
    "hilog:libhilog",
    "hiview:libxpower_event_js",
    "napi:ace_napi",
  ]
  relative_install_dir = "module/multimedia"
  // 必选：所属部件名称
  part_name = "audio_framework"
  // 子系统名称
  subsystem_name = "multimedia"
}
```

**依赖group**

audio_service:audio_service_packages的BUILD.gn如下：

```json
pulseaudio_dir = "//third_party/pulseaudio"
pulseaudio_build_path = "//third_party/pulseaudio/ohosbuild"

group("audio_service_packages") {
  deps = [
    ":audio_common",
    ":audio_service",
    ":audio_service_init",
  ]
}

ohos_shared_library("audio_common") {
  ... // 略
  part_name = "audio_framework"
}

ohos_prebuilt_etc("audio_service_init") {
  ... // 略
  part_name = "audio_framework"
}

ohos_shared_library("audio_process_service") {
  ... // 略
  part_name = "audio_framework"
}

ohos_shared_library("audio_service") {
  ... // 略
  part_name = "audio_framework"
}
```

##### 通过inner_kits关联到部件

bundle.json:

```json
"inner_kits": [
    {
        "type": "none",
        "name": "//foundation/multimedia/audio_framework/services/audio_service:audio_client",
        "header": {
            "header_files": [
                "audio_system_manager.h",
                "audio_stream_manager.h",
                "audio_info.h"
            ],
            "header_base": [
                "//foundation/multimedia/audio_framework/interfaces/inner_api/native/audiomanager/include",
                "//foundation/multimedia/audio_framework/interfaces/inner_api/native/audiocommon/include"
            ]
        }
    },
]
```

audio_service:audio_service_packages的BUILD.gn中，声明了audio_client的模块：

```json
ohos_shared_library("audio_client") {
  ... // 略
  part_name = "audio_framework"
}
```

##### 通过test关联到部件

bundle.json:

```json
"test": [
    "//foundation/multimedia/audio_framework/services/audio_service:audio_service_test_packages",
    ... // 略
]
```

audio_service:audio_service_packages的BUILD.gn中，声明了audio_service_test_packages的模块：

```json
group("audio_service_test_packages") {
  deps = [
    ":audio_hdi_device_test",
    ":audio_multichannel_test",
    ":audio_process_client_test",
    ":audio_service_playback_test",
    ":audio_service_record_test",
  ]
}

ohos_executable("audio_process_client_test") {
  ... // 略
  part_name = "audio_framework"
  subsystem_name = "multimedia"
}

ohos_executable("audio_hdi_device_test") {
  ... // 略
  part_name = "audio_framework"
}

ohos_executable("audio_service_playback_test") {
  ... // 略
  part_name = "audio_framework"
}

ohos_executable("audio_faststream_playback_test") {
  ... // 略
  part_name = "audio_framework"
}

ohos_executable("audio_service_record_test") {
  ... // 略
  part_name = "audio_framework"
}

ohos_executable("audio_multichannel_test") {
  ... // 略
  part_name = "audio_framework"
}
```

注意到这个group中的每个模块是ohos_executable类型的，即可执行程序。

# 附录：编译子系统

编译工具：gn->ninja->GCC

编译逻辑定义：产品>部件>模块

模块配置规则：https://blog.csdn.net/zhoudidong/article/details/129684830

# 附录：鸿蒙源码

源码下载（非Gitee）：https://repo.huaweicloud.com/harmonyos/os/

鸿蒙内核源码分析：http://weharmonyos.com/weharmony/
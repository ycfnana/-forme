# CREATE_WORKFLOW

上sample：

```
import json

import kfp.compiler as compiler
import kfp.components as components
from kfp import dsl
from kubernetes.client import V1ResourceRequirements

def pipeline_func():
    component = components.load_component_from_file('component.yaml')
    kargs = {
        'a': 1,
      'b': 2,
    }

    param1 = dsl.PipelineParam(name='sda', value='sdad')
    param2 = dsl.PipelineParam(name='sddsaa', value='sdad')

    container_op = component(**kargs)
    group_id = dsl.Condition(container_op.output == 3)
    # group_id = dsl.Condition(param1 == '1')
    with group_id:
        container_op1 = component(**kargs)
        group_id_1 = dsl.Condition(container_op1.output == 3)
        with group_id_1:
            container_op2 = component(**kargs)
            container_op3 = component(**kargs)
            container_op3.after(container_op2)
        group_id_1.after(container_op1)




if __name__ == '__main__':
    workflow = compiler.Compiler().create_workflow(pipeline_func, pipeline_name='sdada')
    print(json.dumps(workflow))
```

所以还是create_workflow函数干了，他需要pipeline_func去生成对应的worflow yaml

先说下pipeline_func，执行改func之后，会生成一个workflow yaml，dsl生成的Condition、container_op都是对应worflow的op，然后根据op对应的after函数，组装workflow的拓扑结构，如下图1：

![](/Users/yuchenfeng/Library/Application%20Support/marktext/images/2022-06-20-20-46-47-image.png)

依次来看create_worflow干了些啥，

![](/Users/yuchenfeng/Library/Application%20Support/marktext/images/2022-06-20-20-47-53-image.png)

他直接执行如下代码：

```
with dsl.Pipeline(pipeline_name) as dsl_pipeline:
  pipeline_func(*args_list, **kwargs_dict)
```

即直接执行我们自定义的pipeline_func。

所以 问题来了，在生成的condition也好，container_op也好，生成的这些变量都没有被用到，但是最终的pipeline中是包含了这些变量的

![](/Users/yuchenfeng/Library/Application%20Support/marktext/images/2022-06-20-21-03-01-image.png)

![](/Users/yuchenfeng/Library/Application%20Support/marktext/images/2022-06-20-21-03-41-image.png)

可以看到最终生成的pipeline值是有的。

所以  他牛逼在 执行的with的时候  干了些自己的事情，

![](/Users/yuchenfeng/Library/Application%20Support/marktext/images/2022-06-20-20-56-28-image.png)

如上图所示为pipeline的enter函数，在执行with时，会执行这个enter方法，他在生成container_op时，会执行一个register_op_and_generate_id方法。

```
  def add_op(self, op: _container_op.BaseOp, define_only: bool):
    """Add a new operator.

    Args:
      op: An operator of ContainerOp, ResourceOp or their inherited types.
        Returns
      op_name: a unique op name.
    """
    # Sanitizing the op name.
    # Technically this could be delayed to the compilation stage, but string
    # serialization of PipelineParams make unsanitized names problematic.
    op_name = _naming._sanitize_python_function_name(op.human_name).replace(
        '_', '-')
    #If there is an existing op with this name then generate a new name.
    op_name = _naming._make_name_unique_by_adding_index(op_name,
                                                        list(self.ops.keys()),
                                                        ' ')
    if op_name == '':
      op_name = _naming._make_name_unique_by_adding_index(
          'task', list(self.ops.keys()), ' ')

    self.ops[op_name] = op
    if not define_only:
      self.groups[-1].ops.append(op)

    return op_name
```

他会把这个container_op放到pipeline的ops，并且会放入groups的堆栈中。所以解释了为什么ops数组里面会有对应container_op。

那么groups存了哪些东西，截图上可以看出他实际上就存了 定义的这些condition，在使用这些condition时，我们都会加上with，所以可以直接看看with condition干了些啥

```
  def __enter__(self) -> _for_loop.LoopArguments:
    _ = super().__enter__()
    return self.loop_args
```

```
  def __enter__(self):
    if not _pipeline.Pipeline.get_default_pipeline():
      raise ValueError('Default pipeline not defined.')

    self.recursive_ref = self._get_matching_opsgroup_already_in_pipeline(
        self.type, self.name)
    if not self.recursive_ref:
      self._make_name_unique()

    _pipeline.Pipeline.get_default_pipeline().push_ops_group(self)
    return self
```

直接看最后一行  他会把把这个condition_op 也push进groups中。

groups可以简单理解为一个堆栈，作用就是确保可以将当前op，放置在上一层的group中。保证层级结构的正确。

# After函数

after函数是为了建立拓扑结构，上述步骤 只是告诉这些变量实际上是被存储起来了，但是要最终创建对应的workflow yaml，需要奖励==建立相应的拓扑结构。

我们先看下argo yaml会长什么样。

```
{
  "apiVersion": "argoproj.io/v1alpha1",
  "kind": "Workflow",
  "metadata": {
    "generateName": "sdada-",
    "annotations": {
      "pipelines.kubeflow.org/kfp_sdk_version": "1.6.2",
      "pipelines.kubeflow.org/pipeline_compilation_time": "2022-06-20T21:17:33.763648",
      "pipelines.kubeflow.org/pipeline_spec": "{\"name\": \"sdada\"}"
    },
    "labels": {
      "pipelines.kubeflow.org/kfp_sdk_version": "1.6.2"
    }
  },
  "spec": {
    "entrypoint": "sdada",
    "templates": [
      {
        "name": "add",
        "container": {
          "args": [
            "--a",
            "1",
            "--b",
            "2",
            "----output-paths",
            "/tmp/outputs/Output/data"
          ],
          "command": [
            "sh",
            "-ec",
            "program_path=$(mktemp)\nprintf \"%s\" \"$0\" > \"$program_path\"\npython3 -u \"$program_path\" \"$@\"\n",
            "def add(a, b):\n  '''Calculates sum of two arguments'''\n  return a + b\n\ndef _serialize_float(float_value: float) -> str:\n    if isinstance(float_value, str):\n        return float_value\n    if not isinstance(float_value, (float, int)):\n        raise TypeError('Value \"{}\" has type \"{}\" instead of float.'.format(\n            str(float_value), str(type(float_value))))\n    return str(float_value)\n\nimport argparse\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\n_parser.add_argument(\"--a\", dest=\"a\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"--b\", dest=\"b\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"----output-paths\", dest=\"_output_paths\", type=str, nargs=1)\n_parsed_args = vars(_parser.parse_args())\n_output_files = _parsed_args.pop(\"_output_paths\", [])\n\n_outputs = add(**_parsed_args)\n\n_outputs = [_outputs]\n\n_output_serializers = [\n    _serialize_float,\n\n]\n\nimport os\nfor idx, output_file in enumerate(_output_files):\n    try:\n        os.makedirs(os.path.dirname(output_file))\n    except OSError:\n        pass\n    with open(output_file, 'w') as f:\n        f.write(_output_serializers[idx](_outputs[idx]))\n"
          ],
          "image": "python:3.7"
        },
        "outputs": {
          "parameters": [
            {
              "name": "add-Output",
              "valueFrom": {
                "path": "/tmp/outputs/Output/data"
              }
            }
          ],
          "artifacts": [
            {
              "name": "add-Output",
              "path": "/tmp/outputs/Output/data"
            }
          ]
        },
        "metadata": {
          "labels": {
            "pipelines.kubeflow.org/kfp_sdk_version": "1.6.2",
            "pipelines.kubeflow.org/pipeline-sdk-type": "kfp"
          },
          "annotations": {
            "pipelines.kubeflow.org/component_spec": "{\"description\": \"Calculates sum of two arguments\", \"implementation\": {\"container\": {\"args\": [\"--a\", {\"inputValue\": \"a\"}, \"--b\", {\"inputValue\": \"b\"}, \"----output-paths\", {\"outputPath\": \"Output\"}], \"command\": [\"sh\", \"-ec\", \"program_path=$(mktemp)\\nprintf \\\"%s\\\" \\\"$0\\\" > \\\"$program_path\\\"\\npython3 -u \\\"$program_path\\\" \\\"$@\\\"\\n\", \"def add(a, b):\\n  '''Calculates sum of two arguments'''\\n  return a + b\\n\\ndef _serialize_float(float_value: float) -> str:\\n    if isinstance(float_value, str):\\n        return float_value\\n    if not isinstance(float_value, (float, int)):\\n        raise TypeError('Value \\\"{}\\\" has type \\\"{}\\\" instead of float.'.format(\\n            str(float_value), str(type(float_value))))\\n    return str(float_value)\\n\\nimport argparse\\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\\n_parser.add_argument(\\\"--a\\\", dest=\\\"a\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"--b\\\", dest=\\\"b\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"----output-paths\\\", dest=\\\"_output_paths\\\", type=str, nargs=1)\\n_parsed_args = vars(_parser.parse_args())\\n_output_files = _parsed_args.pop(\\\"_output_paths\\\", [])\\n\\n_outputs = add(**_parsed_args)\\n\\n_outputs = [_outputs]\\n\\n_output_serializers = [\\n    _serialize_float,\\n\\n]\\n\\nimport os\\nfor idx, output_file in enumerate(_output_files):\\n    try:\\n        os.makedirs(os.path.dirname(output_file))\\n    except OSError:\\n        pass\\n    with open(output_file, 'w') as f:\\n        f.write(_output_serializers[idx](_outputs[idx]))\\n\"], \"image\": \"python:3.7\"}}, \"inputs\": [{\"name\": \"a\", \"type\": \"Float\"}, {\"name\": \"b\", \"type\": \"Float\"}], \"name\": \"Add\", \"outputs\": [{\"name\": \"Output\", \"type\": \"Float\"}]}",
            "pipelines.kubeflow.org/component_ref": "{\"digest\": \"b085aca281d941770a9e9c1d066ed481788012a968702f5aa7bee053e50c9e98\", \"url\": \"component.yaml\"}",
            "pipelines.kubeflow.org/arguments.parameters": "{\"a\": \"1\", \"b\": \"2\"}"
          }
        }
      },
      {
        "name": "add-2",
        "container": {
          "args": [
            "--a",
            "1",
            "--b",
            "2",
            "----output-paths",
            "/tmp/outputs/Output/data"
          ],
          "command": [
            "sh",
            "-ec",
            "program_path=$(mktemp)\nprintf \"%s\" \"$0\" > \"$program_path\"\npython3 -u \"$program_path\" \"$@\"\n",
            "def add(a, b):\n  '''Calculates sum of two arguments'''\n  return a + b\n\ndef _serialize_float(float_value: float) -> str:\n    if isinstance(float_value, str):\n        return float_value\n    if not isinstance(float_value, (float, int)):\n        raise TypeError('Value \"{}\" has type \"{}\" instead of float.'.format(\n            str(float_value), str(type(float_value))))\n    return str(float_value)\n\nimport argparse\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\n_parser.add_argument(\"--a\", dest=\"a\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"--b\", dest=\"b\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"----output-paths\", dest=\"_output_paths\", type=str, nargs=1)\n_parsed_args = vars(_parser.parse_args())\n_output_files = _parsed_args.pop(\"_output_paths\", [])\n\n_outputs = add(**_parsed_args)\n\n_outputs = [_outputs]\n\n_output_serializers = [\n    _serialize_float,\n\n]\n\nimport os\nfor idx, output_file in enumerate(_output_files):\n    try:\n        os.makedirs(os.path.dirname(output_file))\n    except OSError:\n        pass\n    with open(output_file, 'w') as f:\n        f.write(_output_serializers[idx](_outputs[idx]))\n"
          ],
          "image": "python:3.7"
        },
        "outputs": {
          "parameters": [
            {
              "name": "add-2-Output",
              "valueFrom": {
                "path": "/tmp/outputs/Output/data"
              }
            }
          ],
          "artifacts": [
            {
              "name": "add-2-Output",
              "path": "/tmp/outputs/Output/data"
            }
          ]
        },
        "metadata": {
          "labels": {
            "pipelines.kubeflow.org/kfp_sdk_version": "1.6.2",
            "pipelines.kubeflow.org/pipeline-sdk-type": "kfp"
          },
          "annotations": {
            "pipelines.kubeflow.org/component_spec": "{\"description\": \"Calculates sum of two arguments\", \"implementation\": {\"container\": {\"args\": [\"--a\", {\"inputValue\": \"a\"}, \"--b\", {\"inputValue\": \"b\"}, \"----output-paths\", {\"outputPath\": \"Output\"}], \"command\": [\"sh\", \"-ec\", \"program_path=$(mktemp)\\nprintf \\\"%s\\\" \\\"$0\\\" > \\\"$program_path\\\"\\npython3 -u \\\"$program_path\\\" \\\"$@\\\"\\n\", \"def add(a, b):\\n  '''Calculates sum of two arguments'''\\n  return a + b\\n\\ndef _serialize_float(float_value: float) -> str:\\n    if isinstance(float_value, str):\\n        return float_value\\n    if not isinstance(float_value, (float, int)):\\n        raise TypeError('Value \\\"{}\\\" has type \\\"{}\\\" instead of float.'.format(\\n            str(float_value), str(type(float_value))))\\n    return str(float_value)\\n\\nimport argparse\\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\\n_parser.add_argument(\\\"--a\\\", dest=\\\"a\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"--b\\\", dest=\\\"b\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"----output-paths\\\", dest=\\\"_output_paths\\\", type=str, nargs=1)\\n_parsed_args = vars(_parser.parse_args())\\n_output_files = _parsed_args.pop(\\\"_output_paths\\\", [])\\n\\n_outputs = add(**_parsed_args)\\n\\n_outputs = [_outputs]\\n\\n_output_serializers = [\\n    _serialize_float,\\n\\n]\\n\\nimport os\\nfor idx, output_file in enumerate(_output_files):\\n    try:\\n        os.makedirs(os.path.dirname(output_file))\\n    except OSError:\\n        pass\\n    with open(output_file, 'w') as f:\\n        f.write(_output_serializers[idx](_outputs[idx]))\\n\"], \"image\": \"python:3.7\"}}, \"inputs\": [{\"name\": \"a\", \"type\": \"Float\"}, {\"name\": \"b\", \"type\": \"Float\"}], \"name\": \"Add\", \"outputs\": [{\"name\": \"Output\", \"type\": \"Float\"}]}",
            "pipelines.kubeflow.org/component_ref": "{\"digest\": \"b085aca281d941770a9e9c1d066ed481788012a968702f5aa7bee053e50c9e98\", \"url\": \"component.yaml\"}",
            "pipelines.kubeflow.org/arguments.parameters": "{\"a\": \"1\", \"b\": \"2\"}"
          }
        }
      },
      {
        "name": "add-3",
        "container": {
          "args": [
            "--a",
            "1",
            "--b",
            "2",
            "----output-paths",
            "/tmp/outputs/Output/data"
          ],
          "command": [
            "sh",
            "-ec",
            "program_path=$(mktemp)\nprintf \"%s\" \"$0\" > \"$program_path\"\npython3 -u \"$program_path\" \"$@\"\n",
            "def add(a, b):\n  '''Calculates sum of two arguments'''\n  return a + b\n\ndef _serialize_float(float_value: float) -> str:\n    if isinstance(float_value, str):\n        return float_value\n    if not isinstance(float_value, (float, int)):\n        raise TypeError('Value \"{}\" has type \"{}\" instead of float.'.format(\n            str(float_value), str(type(float_value))))\n    return str(float_value)\n\nimport argparse\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\n_parser.add_argument(\"--a\", dest=\"a\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"--b\", dest=\"b\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"----output-paths\", dest=\"_output_paths\", type=str, nargs=1)\n_parsed_args = vars(_parser.parse_args())\n_output_files = _parsed_args.pop(\"_output_paths\", [])\n\n_outputs = add(**_parsed_args)\n\n_outputs = [_outputs]\n\n_output_serializers = [\n    _serialize_float,\n\n]\n\nimport os\nfor idx, output_file in enumerate(_output_files):\n    try:\n        os.makedirs(os.path.dirname(output_file))\n    except OSError:\n        pass\n    with open(output_file, 'w') as f:\n        f.write(_output_serializers[idx](_outputs[idx]))\n"
          ],
          "image": "python:3.7"
        },
        "outputs": {
          "artifacts": [
            {
              "name": "add-3-Output",
              "path": "/tmp/outputs/Output/data"
            }
          ]
        },
        "metadata": {
          "labels": {
            "pipelines.kubeflow.org/kfp_sdk_version": "1.6.2",
            "pipelines.kubeflow.org/pipeline-sdk-type": "kfp"
          },
          "annotations": {
            "pipelines.kubeflow.org/component_spec": "{\"description\": \"Calculates sum of two arguments\", \"implementation\": {\"container\": {\"args\": [\"--a\", {\"inputValue\": \"a\"}, \"--b\", {\"inputValue\": \"b\"}, \"----output-paths\", {\"outputPath\": \"Output\"}], \"command\": [\"sh\", \"-ec\", \"program_path=$(mktemp)\\nprintf \\\"%s\\\" \\\"$0\\\" > \\\"$program_path\\\"\\npython3 -u \\\"$program_path\\\" \\\"$@\\\"\\n\", \"def add(a, b):\\n  '''Calculates sum of two arguments'''\\n  return a + b\\n\\ndef _serialize_float(float_value: float) -> str:\\n    if isinstance(float_value, str):\\n        return float_value\\n    if not isinstance(float_value, (float, int)):\\n        raise TypeError('Value \\\"{}\\\" has type \\\"{}\\\" instead of float.'.format(\\n            str(float_value), str(type(float_value))))\\n    return str(float_value)\\n\\nimport argparse\\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\\n_parser.add_argument(\\\"--a\\\", dest=\\\"a\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"--b\\\", dest=\\\"b\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"----output-paths\\\", dest=\\\"_output_paths\\\", type=str, nargs=1)\\n_parsed_args = vars(_parser.parse_args())\\n_output_files = _parsed_args.pop(\\\"_output_paths\\\", [])\\n\\n_outputs = add(**_parsed_args)\\n\\n_outputs = [_outputs]\\n\\n_output_serializers = [\\n    _serialize_float,\\n\\n]\\n\\nimport os\\nfor idx, output_file in enumerate(_output_files):\\n    try:\\n        os.makedirs(os.path.dirname(output_file))\\n    except OSError:\\n        pass\\n    with open(output_file, 'w') as f:\\n        f.write(_output_serializers[idx](_outputs[idx]))\\n\"], \"image\": \"python:3.7\"}}, \"inputs\": [{\"name\": \"a\", \"type\": \"Float\"}, {\"name\": \"b\", \"type\": \"Float\"}], \"name\": \"Add\", \"outputs\": [{\"name\": \"Output\", \"type\": \"Float\"}]}",
            "pipelines.kubeflow.org/component_ref": "{\"digest\": \"b085aca281d941770a9e9c1d066ed481788012a968702f5aa7bee053e50c9e98\", \"url\": \"component.yaml\"}",
            "pipelines.kubeflow.org/arguments.parameters": "{\"a\": \"1\", \"b\": \"2\"}"
          }
        }
      },
      {
        "name": "add-4",
        "container": {
          "args": [
            "--a",
            "1",
            "--b",
            "2",
            "----output-paths",
            "/tmp/outputs/Output/data"
          ],
          "command": [
            "sh",
            "-ec",
            "program_path=$(mktemp)\nprintf \"%s\" \"$0\" > \"$program_path\"\npython3 -u \"$program_path\" \"$@\"\n",
            "def add(a, b):\n  '''Calculates sum of two arguments'''\n  return a + b\n\ndef _serialize_float(float_value: float) -> str:\n    if isinstance(float_value, str):\n        return float_value\n    if not isinstance(float_value, (float, int)):\n        raise TypeError('Value \"{}\" has type \"{}\" instead of float.'.format(\n            str(float_value), str(type(float_value))))\n    return str(float_value)\n\nimport argparse\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\n_parser.add_argument(\"--a\", dest=\"a\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"--b\", dest=\"b\", type=float, required=True, default=argparse.SUPPRESS)\n_parser.add_argument(\"----output-paths\", dest=\"_output_paths\", type=str, nargs=1)\n_parsed_args = vars(_parser.parse_args())\n_output_files = _parsed_args.pop(\"_output_paths\", [])\n\n_outputs = add(**_parsed_args)\n\n_outputs = [_outputs]\n\n_output_serializers = [\n    _serialize_float,\n\n]\n\nimport os\nfor idx, output_file in enumerate(_output_files):\n    try:\n        os.makedirs(os.path.dirname(output_file))\n    except OSError:\n        pass\n    with open(output_file, 'w') as f:\n        f.write(_output_serializers[idx](_outputs[idx]))\n"
          ],
          "image": "python:3.7"
        },
        "outputs": {
          "artifacts": [
            {
              "name": "add-4-Output",
              "path": "/tmp/outputs/Output/data"
            }
          ]
        },
        "metadata": {
          "labels": {
            "pipelines.kubeflow.org/kfp_sdk_version": "1.6.2",
            "pipelines.kubeflow.org/pipeline-sdk-type": "kfp"
          },
          "annotations": {
            "pipelines.kubeflow.org/component_spec": "{\"description\": \"Calculates sum of two arguments\", \"implementation\": {\"container\": {\"args\": [\"--a\", {\"inputValue\": \"a\"}, \"--b\", {\"inputValue\": \"b\"}, \"----output-paths\", {\"outputPath\": \"Output\"}], \"command\": [\"sh\", \"-ec\", \"program_path=$(mktemp)\\nprintf \\\"%s\\\" \\\"$0\\\" > \\\"$program_path\\\"\\npython3 -u \\\"$program_path\\\" \\\"$@\\\"\\n\", \"def add(a, b):\\n  '''Calculates sum of two arguments'''\\n  return a + b\\n\\ndef _serialize_float(float_value: float) -> str:\\n    if isinstance(float_value, str):\\n        return float_value\\n    if not isinstance(float_value, (float, int)):\\n        raise TypeError('Value \\\"{}\\\" has type \\\"{}\\\" instead of float.'.format(\\n            str(float_value), str(type(float_value))))\\n    return str(float_value)\\n\\nimport argparse\\n_parser = argparse.ArgumentParser(prog='Add', description='Calculates sum of two arguments')\\n_parser.add_argument(\\\"--a\\\", dest=\\\"a\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"--b\\\", dest=\\\"b\\\", type=float, required=True, default=argparse.SUPPRESS)\\n_parser.add_argument(\\\"----output-paths\\\", dest=\\\"_output_paths\\\", type=str, nargs=1)\\n_parsed_args = vars(_parser.parse_args())\\n_output_files = _parsed_args.pop(\\\"_output_paths\\\", [])\\n\\n_outputs = add(**_parsed_args)\\n\\n_outputs = [_outputs]\\n\\n_output_serializers = [\\n    _serialize_float,\\n\\n]\\n\\nimport os\\nfor idx, output_file in enumerate(_output_files):\\n    try:\\n        os.makedirs(os.path.dirname(output_file))\\n    except OSError:\\n        pass\\n    with open(output_file, 'w') as f:\\n        f.write(_output_serializers[idx](_outputs[idx]))\\n\"], \"image\": \"python:3.7\"}}, \"inputs\": [{\"name\": \"a\", \"type\": \"Float\"}, {\"name\": \"b\", \"type\": \"Float\"}], \"name\": \"Add\", \"outputs\": [{\"name\": \"Output\", \"type\": \"Float\"}]}",
            "pipelines.kubeflow.org/component_ref": "{\"digest\": \"b085aca281d941770a9e9c1d066ed481788012a968702f5aa7bee053e50c9e98\", \"url\": \"component.yaml\"}",
            "pipelines.kubeflow.org/arguments.parameters": "{\"a\": \"1\", \"b\": \"2\"}"
          }
        }
      },
      {
        "name": "condition-1",
        "dag": {
          "tasks": [
            {
              "name": "add-2",
              "template": "add-2"
            },
            {
              "name": "condition-2",
              "template": "condition-2",
              "when": "\"{{tasks.add-2.outputs.parameters.add-2-Output}}\" == \"3\"",
              "dependencies": [
                "add-2"
              ]
            }
          ]
        }
      },
      {
        "name": "condition-2",
        "dag": {
          "tasks": [
            {
              "name": "add-3",
              "template": "add-3"
            },
            {
              "name": "add-4",
              "template": "add-4",
              "dependencies": [
                "add-3"
              ]
            }
          ]
        }
      },
      {
        "name": "sdada",
        "dag": {
          "tasks": [
            {
              "name": "add",
              "template": "add"
            },
            {
              "name": "condition-1",
              "template": "condition-1",
              "when": "\"{{tasks.add.outputs.parameters.add-Output}}\" == \"3\"",
              "dependencies": [
                "add"
              ]
            }
          ]
        }
      }
    ],
    "arguments": {
      "parameters": []
    },
    "serviceAccountName": "pipeline-runner"
  }
}
```

直接看dag字段，它主要在两块，dependencies和task，这两个变量建立了一个拓扑，

![](/Users/yuchenfeng/Library/Application%20Support/marktext/images/2022-06-20-21-14-12-image.png)

他的引用也没有多少，可以挨个来看

```
# ops group
  def after(self, *ops):
    """Specify explicit dependency on other ops."""
    for op in ops:
      self.dependencies.append(op)
    return self
```

```
  # base op （container op）
  def after(self, *ops):
    """Specify explicit dependency on other ops."""
    for op in ops:
      self.dependent_names.append(op.name)
    return self
```

哈哈  就是加到dependencies里面去

就是这样  看懂了with一切都好说。

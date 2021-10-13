[TOC]

# Component model
This section defines component model.

Components describe functional units that may be instantiated as part of a larger distributed application. The [`Application`](application.md) section will describe how components are grouped together and how instances of those components are then configured, while this section will focus on 
component model itself.

## Component Definition

The role of a `ComponentDefinition` entity is to permit component providers to declare, in infrastructure-neutral format, the runtime characteristics of such unit of execution. 

For example, each microservice in an application is described as a component. Note that `ComponentDefinition` itself is *NOT* an instance of that microservice, but a declaration of the configurable attributes of that microservice. These configurable attributes should be expose as a list of parameters which allow the application team to set and instantiate this component later at deployment time.

In practice, a simple containerized workload, a Helm chart, or a cloud database may all be modeled as a component.

### Top-Level Attributes

Here are the attributes that provide top-level information about the component definition.

| Attribute | Type | Required | Default Value | Description |
|-----------|------|----------|---------------|-------------|
| `apiVersion` | `string` | Y | | A string that identifies the version of the schema the object should have. The core types uses `aam.globalsphare.com/v1beta1` in this version of model |
| `kind` | `string` | Y || Must be `ComponentDefinition` |
| `metadata` | [`Metadata`](#metadata) | Y | | Entity metadata. |
| `spec`| [`Spec`](#spec) | Y | | The specification for the component definition. |

#### Metadata

This metadata section is made up of several top-level keys.

Metadata provides information about the contents of the object.

| Attribute | Type | Required | Default Value | Description |
|-----------|------|----------|---------------|-------------|
| `name` | `string` | Y | | A name for the schematic. `name` is subject to the restrictions listed beneath this table. |
| `labels` | `map[string]string` | N | | A set of string key/value pairs used as arbitrary labels on this component. See the "Label format" section immediately below. |
| `annotations` | `map[string]string`| N || A set of string key/value pairs used as arbitrary descriptive text associated with this object. See the "Annotations format" section immediately below. |


#### Spec

| Attribute | Type | Required | Default Value | Description |
|-----------|------|----------|---------------|-------------|
| `schematic` | [Schematic](#schematic) | Y | | Schematic information for this component. |

##### Schematic

This section declares the schematic of a component that could be instantiated as part of an application in the later deployment workflow. Note that AAM itself has no enforcement on how to implement the schematic as long as it could:
  1. model a deployable unit;
  2. expose a JSON schema or equivalent parameter list. 

In Island, `cue` are supported for now.

###### Example

Below is a full example of CUE based component definition named `webserver`:

<p>
<details>

```yaml
apiVersion: aam.globalsphare.com/v1beta1
kind: ComponentDefinition
metadata:
  name: webserver
spec:
  schematic:
    cue:
      template: |
        output: {
            apiVersion: "apps/v1"
            kind:       "Deployment"
            spec: {
                selector: matchLabels: {
                    "app": context.name
                }
                template: {
                    metadata: labels: {
                        "app": context.name
                    }
                    spec: {
                        containers: [{
                            name:  context.name
                            image: parameter.image

                            if parameter["cmd"] != _|_ {
                                command: parameter.cmd
                            }

                            if parameter["env"] != _|_ {
                                env: parameter.env
                            }

                            if context["config"] != _|_ {
                                env: context.config
                            }

                            ports: [{
                                containerPort: parameter.port
                            }]

                            if parameter["cpu"] != _|_ {
                                resources: {
                                    limits:
                                        cpu: parameter.cpu
                                    requests:
                                        cpu: parameter.cpu
                                }
                            }
                        }]
                }
                }
            }
        }
        // an extra template
        outputs: service: {
            apiVersion: "v1"
            kind:       "Service"
            spec: {
                selector: {
                    "app": context.name
                }
                ports: [
                    {
                        port:       parameter.port
                        targetPort: parameter.port
                    },
                ]
            }
        }
        parameter: {
            image: string
            cmd?: [...string]
            port: *80 | int
            env?: [...{
                name:   string
                value?: string
                valueFrom?: {
                    secretKeyRef: {
                        name: string
                        key:  string
                    }
                }
            }]
            cpu?: string
        }
```

</details>
</p>

With above `webserver` installed in the platform, user would be able to deploy this component in an application as below:

```yaml
apiVersion: aam.globalsphare.com/v1beta1
kind: Application
metadata:
  name: webserver-demo
spec:
  components:
    - name: hello-world
      type: webserver               # claim to deploy webserver component definition
      properties:                   # setting parameter values
        image: crccheck/hello-world
        port: 8000                  # this port will be automatically exposed to public
        env:
        - name: "foo"
          value: "bar"
        cpu: "100m"
```
# Deployment com o Watson Machine Learning


### Deployment using Python API  

The complete script can be fond on our [example reposistory](https://github.com/MLOPsStudyGroup/dvc-gitactions/blob/master/src/scripts/Pipelines/model_deploy_pipeline.py), it recives the path to the trained model, the path to the root of the project containing the ```metadata.yaml``` file, and the credencials file.

```python
python3 model_deploy_pipeline.py ./model_file ../path/to/project/ ../credentials.yaml
```

1. Using these arguments constant variables are set
```python
import os
import sys
import yaml

MODEL_PATH = os.path.abspath(sys.argv[1])
PROJ_PATH = os.path.abspath(sys.argv[2])
CRED_PATH = os.path.abspath(sys.argv[3])
META_PATH = PROJ_PATH + "/metadata.yaml"
```
2. After that, the ```yaml``` files are loaded as dictionaries and the model is loaded (using either ```joblib``` or ```pickle```).
```python
import pickle
import joblib

with open(CRED_PATH) as stream:
    try:
        credentials = yaml.safe_load(stream)
    except yaml.YAMLError as exc:
        print(exc)


with open(META_PATH) as stream:
    try:
        metadata = yaml.safe_load(stream)
    except yaml.YAMLError as exc:
        print(exc)

with open(MODEL_PATH, "rb") as file:
    # pickle_model = pickle.load(file)
    pipeline = joblib.load(file)
```
3. The next step is to create an instance of the IBM Watson ```client``` , to do that the credential loaded above will be used and a default [Deployment Space](paginaDoDeploySpace) will be set using the ID contained in the credenciais file, other constants will be set with informations regarding the model found on the ```metadata``` file.
```python
from ibm_watson_machine_learning import APIClient

wml_credentials = {"url": credentials["url"], "apikey": credentials["apikey"]}

client = APIClient(wml_credentials)
client.spaces.list()

MODEL_NAME = metadata["project_name"] + "_" + metadata["project_version"]
DEPLOY_NAME = MODEL_NAME + "-Deployment"
MODEL = pipeline
SPACE_ID = credentials["space_id"]

client.set.default_space(SPACE_ID)
```
4. Before the deployment, we need to give Watson some characteristics from our model, such as name, type (in this case is  ```scikit-learn_0.23```) and specifications of the instance that will run the micro-service. Next, the model is stored as an ```Asset``` on Watson ML.
```python
model_props = {
    client.repository.ModelMetaNames.NAME: MODEL_NAME,
    client.repository.ModelMetaNames.TYPE: metadata["model_type"],
    client.repository.ModelMetaNames.SOFTWARE_SPEC_UID: client.software_specifications.get_id_by_name(
        "default_py3.7"
    ),
}

model_details = client.repository.store_model(model=MODEL, meta_props=model_props)
model_uid = client.repository.get_model_uid(model_details)
```
5. Once complete, we'll give it the deployment a name and then deploy the model using the store ID é feito a partir do ID de armazenamento obobtained in the previous step.
```python
deployment_props = {
    client.deployments.ConfigurationMetaNames.NAME: DEPLOY_NAME,
    client.deployments.ConfigurationMetaNames.ONLINE: {},
}

deployment = client.deployments.create(
    artifact_uid=model_uid, meta_props=deployment_props
)
```
6. Finally, the storage and deployment IDs are added or updated to in the ```metadata.yaml``` file.
```python
deployment_uid = client.deployments.get_uid(deployment)

metadata["model_uid"] = model_uid
metadata["deployment_uid"] = deployment_uid

f = open(META_PATH, "w+")
yaml.dump(metadata, f, allow_unicode=True)
```


### Accessing Model Predictions

Havind deployed the model. we can access it's predictions by sending requests to an end-point or by using the Python ```ibm_watson_machine_learning``` library, where we can send either features for a single prediction or payloads containing multiple lines of a dataframe, for example.

The payload body is made of the dataframe column names under the ```"fields"``` key and the features under ``` "values"``` .

```python
payload = {
    "input_data": [
        {
            "fields": X.columns.to_numpy().tolist(),
            "values": X.to_numpy().tolist(),
        }
    ]
}

result = client.deployments.score(DEPLOYMENT_UID, payload)
```
The model response will contain the scoring result containing prediction and coresponding probability. In the case of a binary classifier, the response will have the following format:

        [1, [0.06057910314628456, 0.9394208968537154]],
        [1, [0.23434887273340754, 0.7656511272665925]],
        [1, [0.08054183674380211, 0.9194581632561979]],
        [1, [0.07877206037184215, 0.9212279396281579]],
        [0, [0.5719774367794239, 0.42802256322057614]],
        [1, [0.017282880299552716, 0.9827171197004473]],
        [1, [0.01714869904990468, 0.9828513009500953]],
        [1, [0.23952044576217457, 0.7604795542378254]],
        [1, [0.03055527110545664, 0.9694447288945434]],
        [1, [0.2879899631347379, 0.7120100368652621]],
        [0, [0.9639766912352016, 0.03602330876479841]],
        [1, [0.049694416576558154, 0.9503055834234418]],


### Updating the Model
Updating the asset contaning the model and/or updating the deployment. The complete scripts for the [deployment](https://github.com/MLOPsStudyGroup/dvc-gitactions/blob/master/src/scripts/Pipelines/model_update_deployment_pipeline.py) and [model](https://github.com/MLOPsStudyGroup/dvc-gitactions/blob/master/src/scripts/Pipelines/model_update_pipeline.py) can be found on our tamplate repository.

1. Firstly we need to update the model asset in WML by passing the new model as well as a name.

```python 
print("\nCreating new version")

published_model = client.repository.update_model(
    model_uid=MODEL_GUID,
    update_model=model,
    updated_meta_props={
        client.repository.ModelMetaNames.NAME: metadata["project_name"]
        + "_"
        + metadata["project_version"]
    },
)
```
2. After that a new revision can be created.
```python 
new_model_revision = client.repository.create_model_revision(MODEL_GUID)

rev_id = new_model_revision["metadata"].get("rev")
print("\nVersion:", rev_id)

client.repository.list_models_revisions(MODEL_GUID)
```
3. Finally we can update the deployment.
```python
change_meta = {client.deployments.ConfigurationMetaNames.ASSET: {"id": MODEL_GUID}}

print("Updating the following model: ")
print(client.deployments.get_details(DEPLOYMENT_UID))

client.deployments.update(DEPLOYMENT_UID, change_meta)
```

### Model Rollback
1. We have previously created revisions of a model, to rollback the model version, we'll list all the revisions made:
```python
client.repository.list_models_revisions(MODEL_GUID)
```
![](../assets/versions.PNG)

2. Now we can choose wich revision we want to rollback to and then update the deployment referencing that revision ID.
```python
MODEL_VERSION = input("MODEL VERSION: ")

meta = {
    client.deployments.ConfigurationMetaNames.ASSET: {
        "id": MODEL_GUID,
        "rev": MODEL_VERSION,
    }
}
updated_deployment = client.deployments.update(
    deployment_uid=DEPLOYMENT_UID, changes=meta
)
```
3. Finally we'll wait for the uptade to finish so we can see if it was successful.
```python
status = None
while status not in ["ready", "failed"]:
    print(".", end=" ")
    time.sleep(2)
    deployment_details = client.deployments.get_details(DEPLOYMENT_UID)
    status = deployment_details["entity"]["status"].get("state")

print("\nDeployment update finished with status: ", status)
```
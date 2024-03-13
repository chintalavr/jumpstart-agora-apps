# Deployment of OpenVino Model Server (OVMS)

[OpenVino Model Server](https://docs.openvino.ai/2024/ovms_what_is_openvino_model_server.html) hosts models and makes them accessible to software components over standard network protocols: a client sends a request to the model server, which performs model inference and sends a response back to the client.

<img alt="OpenVino Model Server" src="https://docs.openvino.ai/2024/_images/ovms_diagram.png" width="800" />

## Node setup
These instructions are for configuring OVMS for AKS Edge Essentials. (Note that the Ubuntu + K3s deployment will be covered separately.) To set up the node correctly, you'll need to copy the [ovms-setup.sh](../scripts/ovms-setup.sh) script to the Linux node and then run it. This script performs the following actions:

1. Create the models repository under `/var/lib/rancher/k3s/storage/models`
1. Download three models (`horizontal-text-detection`, `weld-porosity-detection` and `resnet-50`)
1. Create the model hosting configuration `config.json` file
1. Installing [OpenVino Toolkit Operator](https://docs.openvino.ai/2022.3/openvino_docs_install_guides_overview.html)

To setup the node, run the following steps:

1. Copy the `config.json` file to the AKS-EE node
    ```powershell
    Copy-AksEdgeNodeFile -NodeType Linux -FromFile ..\scripts\ovms-setup.sh -ToFile /home/aksedge-user/ovms-setup.sh -PushFile
    ```
1. Run the `ovms-setup.sh`
    ```powershell
    Invoke-AksEdgeNodeCommand -NodeType Linux -command "sudo sh /home/aksedge-user/ovms-setup.sh"
    ```
1. Check that the three models are downloaded
    ```powershell
    Invoke-AksEdgeNodeCommand -NodeType Linux -command "sudo ls -la /var/lib/rancher/k3s/storage/models"
    ```
    You should see something similar to the following:
    ```bash
    PS C:\jumpstart-agora-apps\contoso_manufacturing\deployment> Invoke-AksEdgeNodeCommand -NodeType Linux -command "sudo ls -la /var/lib/rancher/k3s/storage/models"
    total 24
    drwxr-x--- 3 root root 4096 Mar 11 19:33 horizontal-text-detection
    drwxr-x--- 3 root root 4096 Mar 11 19:33 resnet-50
    drwxr-x--- 3 root root 4096 Mar 11 19:33 weld-porosity-detection
    ```

## OVMS Deployment

The following istructions will deploy OVMS using a Persistent Volume Claim (PVC) and mounts the models that were previously downloaded as part of [Node setup](#node-setup) steps.


1. Create the **ovms-config** configMap
    ```powershell
    kubectl create configmap ovms-config --from-file=.\configs\config.json
    ```

1. Apply the [ovms-setup.yml](./yamls/ovms-setup.yml)
    ```powershell
    kubectl apply -f .\yamls\ovms-setup.yml
    ```
1. If everything was correctly deployed, you should see the following
    ```powershell
    PS C:\jumpstart-agora-apps\contoso_manufacturing\deployment> kubectl get pvc
    NAME              STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    models-path-pvc   Bound    models-path-pv   1Gi        RWO            local-path     7s

    PS C:\jumpstart-agora-apps\contoso_manufacturing\deployment> kubectl get pv
    NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS   REASON   AGE
    models-path-pv   1Gi        RWO            Retain           Bound    default/models-path-pvc   local-path              43s
    
    PS C:\jumpstart-agora-apps\contoso_manufacturing\deployment> kubectl get pods
    NAME                        READY   STATUS    RESTARTS   AGE
    ovms-sample-8b68dc5-g59sd   1/1     Running   0          14s
    ```
1. Check the **ovms-sample** service IP
    ```powershell
    PS C:\Users\jumpstart-agora-apps\contoso_manufacturing\deployment> kubectl get svc
    NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
    kubernetes    ClusterIP   10.43.0.1      <none>        443/TCP             29m
    ovms-sample   ClusterIP   10.43.84.192   <none>        8080/TCP,8081/TCP   2m55s
    ```

1. Check that models are being loaded correctly
    ```powershell
    Invoke-AksEdgeNodeCommand -NodeType Linux -command "curl http://<ovms-service-ip>:8081/v1/config"
    ```

    If all models are loaded correctly, you should see something similir to the following:
    ```powershell
    {
        "horizontal-text-detection": 
        {
            "model_version_status": [
                {
                    "version": "1",
                    "state": "AVAILABLE",
                    "status": {
                        "error_code": "OK",
                        "error_message": "OK"
                    }
                }
            ]
        },
        "resnet-50": 
        {
            "model_version_status": [
                {
                    "version": "1",
                    "state": "AVAILABLE",
                    "status": {
                        "error_code": "OK",
                        "error_message": "OK"
                    }
                }
            ]
        },
        "weld-porosity-detection":
        {
            "model_version_status": [
                {
                    "version": "1",
                    "state": "AVAILABLE",
                    "status": {
                        "error_code": "OK",
                        "error_message": "OK"
                    }
                }
            ]
        }
    } 
    ```

## Check OVMS inferencing

The final step in this guide is to verify the correct functionality of the OVMS inferencing server by sending an image to the OVMS REST API and examining the resulting inferencing output.

1. Create a new Pod deployment for running a Python sample app
    ```powershell
    kubectl create deployment client-test --image=python:3.8.13 -- sleep infinity
    ```
1. Wait until the Python Pod is running and then exec into it
    ```powershell
    kubectl exec -it $(kubectl get pod -o jsonpath="{.items[0].metadata.name}" -l app=client-test) -- /bin/sh
    ```

1. Install the Python dependencies and sample apps inside the Pod
    ```bash
    cd /tmp
    wget https://raw.githubusercontent.com/openvinotoolkit/model_server/releases/2024/0/demos/object_detection/python/object_detection.py
    wget https://raw.githubusercontent.com/openvinotoolkit/model_server/releases/2024/0/demos/object_detection/python/requirements.txt
    wget https://raw.githubusercontent.com/openvinotoolkit/open_model_zoo/master/data/dataset_classes/coco_91cl.txt
    pip install --upgrade pip
    pip install -r requirements.txt
    ```
1. Download the image to test the inference
    ```bash
    wget https://storage.openvinotoolkit.org/repositories/openvino_notebooks/data/data/image/coco_bike.jpg
    ```
    <img alt="Dog test" src="https://storage.openvinotoolkit.org/repositories/openvino_notebooks/data/data/image/coco_bike.jpg" width="400" />

1. Create the `object_detection.py` sample app
    ```bash
    cat << EOF > object_detection.py
    from ovmsclient import make_grpc_client
    import cv2
    import numpy as np
    import argparse
    import random
    from typing import Optional, Dict

    parser = argparse.ArgumentParser(description='Make object detection prediction using images in binary format')
    parser.add_argument('--image', required=True, help='Path to a image in JPG or PNG format')
    parser.add_argument('--service_url', required=False, default='localhost:9000', help='Specify url to grpc service. default:localhost:9000', dest='service_url')
    parser.add_argument('--model_name', default='faster_rcnn', help='Model name to query. default: faster_rcnn')
    parser.add_argument('--labels', default="coco_91cl.txt", type=str, help='Path to COCO dataset labels with human readable class names')
    parser.add_argument('--input_name', default='input_tensor', help='Input name to query. default: input_tensor')
    parser.add_argument('--model_version', default=0, type=int, help='Model version to query. default: latest available')

    args = parser.parse_args()
    image = cv2.imread(filename=str(args.image))
    image = cv2.cvtColor(image, code=cv2.COLOR_BGR2RGB)
    resized_image = cv2.resize(src=image, dsize=(255, 255))
    network_input_image = np.expand_dims(resized_image, 0)

    client = make_grpc_client(args.service_url)
    inputs = { args.input_name: network_input_image }
    response = client.predict(inputs, args.model_name, args.model_version)

    with open(args.labels, "r") as file:
        coco_labels = file.read().strip().split("\n")
        coco_labels_map = dict(enumerate(coco_labels, 1))

    detection_boxes: np.ndarray = response["detection_boxes"]
    detection_classes: np.ndarray = response["detection_classes"]
    detection_scores: np.ndarray = response["detection_scores"]
    num_detections: np.ndarray = response["num_detections"]

    for i in range(100):
        detected_class_name = coco_labels_map[int(detection_classes[0, i])]
        score = detection_scores[0, i]
        label = f"{detected_class_name} {score:.2f}"
        if score > 0.5:
            print(label)
    EOF
    ```

1. Run the **object_detection.py** sample app
    ```bash
    python object_detection.py --image coco_bike.jpg --model_name resnet-50 --service_url ovms-sample:8080
    ```
1. If everything is working correctly, you should see the following
    ```bash
    # python object_detection.py --image coco_bike.jpg --model_name resnet-50 --service_url ovms-sample:8080
    dog 0.98
    bicycle 0.94
    bicycle 0.93
    car 0.88
    bicycle 0.84
    truck 0.59
    bicycle 0.56
    bicycle 0.54
    ```
# TrustyAI Guardrails + LLM Evaluation Hands on

## 1. Pre-requisites
This exercise was tested against:
- Red Hat OpenShift AI - 2.16.2
- Red Hat Authorino Operator - 1.1.2
- Node Feature Discovery Operator - 4.18.0-202503210101
- NVIDIA GPU Operator - 24.9.2

This exercise also requires the LLM to be deployed as a [KServe Raw deployment](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2-latest/html/monitoring_data_science_models/configuring-the-guardrails-orchestrator-service_monitor#deploying-the-guardrails-orchestrator-service_monitor). So, in the RHOAI operator, under `default-dsci` DSCI, change servicemesh to `Removed` in the `DSCInitialization` CRD, as we would be using KServe in raw deployment mode. 

## 2. Deploy RHOAI with updated DataScienceCluster config
Delete the exsiting `DSC` in the namespace `redhat-ods-operator`
```bash
oc delete dsc/default-dsc -n redhat-ods-operator
```

Review the `dsc.yaml` in this repo and apply it.
```bash
oc apply -f dsc.yaml
```
This will use the latest upstream image of TrustyAI. Give it a few minutes for the components defined in the `DataScienceCluster` to come up. 

## 3. Deploy the Qwen2.5-0.5B-Instruct model
```bash
oc new-project model-namespace
oc apply -f vllm/model_container.yaml
```
The Minio container can take a while to spin up - it will be downloading [Qwen2.5-0.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct) from Huggingface and saving it into Minio.

```bash
oc apply -f vllm/serving_runtime.yaml
oc apply -f vllm/isvc_qwen2.yaml
```
Wait for the `qwen2-predictor-XXXXXX` pod to spin up. You can test the model by sending some prompts to it. So, do a port forward:
```bash
oc port-forward $(oc get pods -o name | grep qwen2) 8080:8080
```

And then, in a new terminal tab, try:
```bash
./curl_model 127.0.0.1:8080 "Hello, tell me about yourself"
````

## 4. Guardrails
### 4.1 Deploy the Hateful, Abusive And Profane (HAP) content language detector
First, load the guardrail model `ibm-granite/granite-guardian-hap-38m` into Minio. 
```bash
oc apply -f guardrails/hap_detector/hap_model_container.yaml
```

Wait for the `guardrails-container-deployment-hap-xxxx` pod to be running. Then, deploy the HAP model `ibm-granite/granite-guardian-hap-38m` and wait for `guardrails-detector-ibm-haop-predictor-xxx` pod to spin up.
```bash
oc apply -f guardrails/hap_detector/hap_serving_runtime.yaml
oc apply -f guardrails/hap_detector/hap_isvc.yaml
```

### 4.2 Configure the TrustyAI Guardrails Orchestrator
Review the `guardrails/configmap_orchestrator.yaml`. Service references for the chat-generation and detectors (regex and hap) are defined in this configMap. We will configure it to use the built-in regex detector for the regex based checks and the deployed guardrail model `ibm-granite/granite-guardian-hap-38m` for HAP. 

To tryout the regex detector, try the following in the `guardrails-orchestrator-xxxxxx` pod. 
```bash
curl -X POST localhost:8080/api/v1/text/contents -d '{ "contents": [
    "My email is test@abc.com! My SSN is 123-45-6789."
  ],
  "detector_params": { "regex": ["email", "ssn"] }
}' -H 'content-type: application/json'
```

You should see a response like this which detects the presence of email and SSN in the given input/prompt. 
```bash
[[{"detection":"EmailAddress","detection_type":"pii","end":24,"score":1.0,"start":12,"text":"test@abc.com"},{"detection":"SocialSecurity","detection_type":"pii","end":47,"score":1.0,"start":36,"text":"123-45-6789"}]]
```

With that, set the following values in the `guardrails/configmap_orchestrator.yaml` as follows:
- `chat_generation.service.hostname`: Set this to the name of your Qwen2 predictor service. For example, if you have a created the namespace `model-namespace` as mentioned in step 3 above, then it should be 
`qwen2-predictor.model-namespace.svc.cluster.local`
- `detectors.hap.service.hostname`: Set this to the name of your HAP predictor service. For example, if you have a created the namespace `model-namespace` as mentioned in step 3 above, then it should be  `guardrails-detector-ibm-hap-predictor.model-namespace.svc.cluster.local`

Review `guardrails/configmap_auxiliary_images.yaml`. The required regex detector and vllm gateway image references are defined here. 

Review `guardrails/configmap_vllm_gateway.yaml`. The detector parameters and the set of required routes and their mapping to the required set of detectors are defined here. 

Now, create all these three configMaps.
```bash
oc apply -f guardrails/configmap_auxiliary_images.yaml
oc apply -f guardrails/configmap_orchestrator.yaml
oc apply -f guardrails/configmap_vllm_gateway.yaml
```

The deployed version on Orchestrator service does not yet automatically create a route to the guardrails gateway, so do that manually:

```bash
oc apply -f guardrails/gateway_route.yaml
```

### 4.3. Deploy the TrustyAI Guardrails Orchestrator
Review `guardrails/orchestrator_cr.yaml` and deploy it. 
```bash
oc apply -f guardrails/orchestrator_cr.yaml
```

### 4.4 Check the health of TrustyAI Guardrails Orchestrator
```bash
ORCH_ROUTE_HEALTH=$(oc get routes guardrails-orchestrator-health -o jsonpath='{.spec.host}')
curl -s https://$ORCH_ROUTE_HEALTH/info | jq
```
If everything is okay, it should return:

```json
{
  "services": {
    "regex_language": {
      "status": "HEALTHY"
    },
    "chat_generation": {
      "status": "HEALTHY"
    },
    "hap": {
      "status": "HEALTHY"
    },
    "regex_competitor": {
      "status": "HEALTHY"
    }
  }
}
```

### 4.5 Play around with the Guardrails Orchestrator
First, set up:

```bash
ORCH_GATEWAY=https://$(oc get routes guardrails-gateway -o jsonpath='{.spec.host}')
```

The available endpoints are:

- `$ORCH_GATEWAY/passthrough`: query the raw, unguardrailed model. 
- `$ORCH_GATEWAY/language_quality`: query with filters for personally identifiable information and HAP
- `$ORCH_GATEWAY/all`: query with all available filters, so the language filters plus a check against competitor names. 


Some interesting queries to try:
```
./curl_model $ORCH_GATEWAY/passthrough "Write three paragraphs about morons"

./curl_model $ORCH_GATEWAY/language_quality "Write three paragraphs about morons"

./curl_model $ORCH_GATEWAY/language_quality "My email address is abc@def.com"

./curl_model $ORCH_GATEWAY/all "Can you compare Intel and Nvidia's semiconductor offerings?"

./curl_model $ORCH_GATEWAY/language_quality "Can you compare Intel and Nvidia's semiconductor offerings?"

```


## Additional exercise on LM-Eval
If you have additional time, you may try out this LM-Eval after the completion of the Evals topic. 

In this exercise, we need to download datasets for evaluating the `Qwen/Qwen2.5-0.5B-Instruct` language model. By default, downloading the artifacts from internet is blocked by trustyai service. 

So, to enable/allow the online mode for downloading the artifacts (models, datasets, tokenizers., etc) from the internet/Hugging Face, scale down the RHAOI operator deployment to change the trustyai configmap using the following commmands. 

```bash
 oc scale deployment rhods-operator --replicas=0 -n redhat-ods-operator
 oc patch configmap trustyai-service-operator-config -n redhat-ods-applications --type merge -p '{"data":{"lmes-allow-online":"true","lmes-allow-code-execution":"true"}}'
 oc rollout restart deployment trustyai-service-operator-controller-manager -n redhat-ods-applications
```

Deploy the language model eval job. 
```bash
oc apply -f lm-eval/lm-eval-job.yaml
```
This will download [the Arc dataset](https://huggingface.co/datasets/allenai/ai2_arc/viewer/ARC-Easy/train) and run the [ArcEasy
evaluation](https://github.com/opendatahub-io/lm-evaluation-harness/tree/main/lm_eval/tasks/arc).

You should see an `evaljob` pod spin up in your cluster. This eval job should take ~5 minutes to run. 
Afterwards, you can navigate to the lmevaljob resource (Home -> Search -> Search for "LMEvalJob" -> Click evaljob)-
inside the lmevaljob's YAML you will see the results of the evaluation, e.g.:
```json
      "results": {
        "arc_easy": {
          "alias": "arc_easy",
          "acc,none": 0.6561447811447811,
          "acc_stderr,none": 0.009746660584852454,
          "acc_norm,none": 0.5925925925925926,
          "acc_norm_stderr,none": 0.010082326627832872
        }
```

Additionally, you may also try the LM-Eval task `arc_challenge` as well. 

## References
- [TrustyAI Notes Repo](https://github.com/trustyai-explainability/reference/tree/main)
- [TrustyAI Github](https://github.com/trustyai-explainability)

Credits: Rob Geada (Red Hat TrustyAI Tech Lead) 

# Challenge 2 — Findings

## Root cause

The AWS DevOps Agent investigated the Lambda function `challenge2-broken-fn` and identified a code issue causing every invocation to fail. The function attempted to access `config["value"]`, but the variable `config` was never defined within the code. This resulted in a `NameError` and caused a 100% failure rate across all invocations.

## Fix applied

I updated the Lambda function code by defining the `config` variable before referencing it. After deploying the change and re-running the test, the function executed successfully and returned the expected response.

```python
def handler(event, context):
    config = {"value": "Challenge 2 Fixed"}
    return {"result": config["value"]}
```

The CloudWatch alarm recovered after the successful execution and the Lambda function returned to a healthy state.

## Evidence

* [x] Screenshot 1: DevOps Agent root-cause analysis (`screenshots/ss1.png`)
* [x] Screenshot 2: Successful Lambda execution and recovery (`screenshots/ss2.png`)

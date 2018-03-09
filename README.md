# Boilerplate

```python
import json

import bugsnag
from bugsnag.delivery import Delivery

class SplunkHECDelivery(Delivery):
    def deliver_sessions(self, config, payload):
        self.deliver(config, payload, {"sourcetype": "bugsnag:session"})

    def deliver(self, config, payload, options=None):
        if options is None:
            options = {}

        try:
            payload = json.loads(payload)
        except Exception as e:
            pass

        def request():
            uri = options.pop('endpoint', config.endpoint)
            if "://" not in uri:
                uri = config.get_endpoint()

            api_key = payload.pop("apiKey", config.get("api_key"))

            req_options = {}
            if config.proxy_host:
                req_options["proxies"] = {
                    "https": config.proxy_host,
                    "http": config.proxy_host
                }

            for event in payload.get("events", []):
                response = requests.post(
                    uri,
                    headers={"Authorization": "Splunk %s" % api_key},
                    json={
                        "index": options.get("index", "bugsnag"),
                        "sourcetype": options.get("sourcetype", "bugsnag"),
                        "source": options.get("source", "bugsnag"),
                        "event": event
                    },
                    **req_options
                )

                status = response.status_code
                if "success" in options:
                    success = options["success"]
                else:
                    success = requests.codes.ok

                if status != success:
                    bugsnag.logger.warning(
                        "Delivery to %s failed, status %d" % (uri, status))

        if config.asynchronous:
            Thread(target=request).start()
        else:
            request()
```

# Usage

```python
bugsnag.configure(
    endpoint="<SPLUNK_HEC_URL>",
    api_key="<SPLUNK_HEC_TOKEN>",
    delivery=SplunkHECDelivery(),
    # Other normal bugsnag config here
)
```

# 🔌 Using Garak with the REST Generator

This guide walks you through configuring and running Garak to test a model served over a REST API using the `rest.RestGenerator`. We'll use [ollama](https://github.com/ollama/ollama/blob/main/docs/api.md) as the example target, but you can adapt this for any service with a compatible REST API.

📚 Reference: [REST Generator Docs](https://reference.garak.ai/en/latest/garak.generators.rest.html)

---

## 📦 Step 1: Build the Generator Config File

🔁 **In the local testing** environment all the variables are automatically copied into a file that's bound to the correct container. You can test by running the rest generator command below.


📁 **In real world use**, you need to create a json file that Garak will use to send requests to the REST service. This needs to be stored somewhere accessible inside the container.

Here's a starting point to creating the JSON file:

```json
{
   "rest": {
      "RestGenerator": {
         "name": "example service",
         "uri": "https://example.ai/llm",
         "method": "post",
         "headers": {
            "X-Authorization": "$KEY"
         },
         "req_template_json_object": {
            "text": "$INPUT"
         },
         "response_json": true,
         "response_json_field": "text"
      }
   }
}
```

🧠 $INPUT will be replaced by Garak with the test prompt.

🕵️‍♀️ Need to customize the headers or payload further?

If you’re not sure what your REST service is expecting under the hood, just:

    Hit Ctrl+Shift+I to open Dev Tools and inspect the Network tab during a chat request.

    Or open up Burp Suite, proxy the request, and copy the exact JSON structure and headers being used.

This is especially useful if you're working with a new model being developed that adds session tokens, changes its payload schema, or uses nonstandard headers.

🎯 The goal is to mimic exactly what the REST service expects — headers and all. Once you’ve got that, Garak can run its probes against it like a charm.

---

## ⚙️ Step 2: Run Garak with REST Generator

Run Garak using your custom config:

```bash
garak --model_type rest \
      --model_name Ollama.Local \
      --generator_option_file /app/rest_request.json \
      --probes xss \
      --generations 3 \
      --report_prefix /app/results/$(date +%Y-%m-%dT%H%M%S)- \
      --verbose

```

📝 You can change:

    --model_name to the model you're working with

    --generator_option_file to the container path of the custom json request
    
    --probes xss to other probes (jailbreak, promptinject, etc.)

    --generations 3 to however many times you want to run each prompt

    --report_prefix to control where the report goes


## 🔍 Debugging Tips

    If no results are generated, watch the command output in the garak container for any errors or a stack trace dump.
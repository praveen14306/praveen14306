#!/bin/bash

# Set the API endpoint URL
API_URL="http://example.com/metrics"

# File to store the JSON metrics
METRICS_JSON_FILE="/tmp/metrics.json"

# Function to poll the API and convert metrics to JSON
poll_and_convert() {
  # Poll the API endpoint and get the plain text metrics
  metrics=$(curl -s $API_URL)

  # Initialize an empty JSON structure
  echo "{" > $METRICS_JSON_FILE

  # Declare an associative array to accumulate metrics
  declare -A metric_data

  # Process each line of metrics
  while IFS= read -r line; do
    # Match lines with labels
    if [[ $line =~ ^([a-zA-Z_][a-zA-Z0-9_]*)\{([^}]*)\}[[:space:]]+([0-9.eE+-]+)$ ]]; then
      metric="${BASH_REMATCH[1]}"
      labels="${BASH_REMATCH[2]}"
      value="${BASH_REMATCH[3]}"

      # Convert labels to JSON
      labels_json=$(echo "$labels" | awk -F, '{
        split($1, a, "=");
        gsub(/"/, "", a[2]);
        label_str = "\"" a[1] "\": \"" a[2] "\"";
        for (i=2; i<=NF; i++) {
          split($i, a, "=");
          gsub(/"/, "", a[2]);
          label_str = label_str ", \"" a[1] "\": \"" a[2] "\""
        }
        print label_str
      }')

      # Append metric to the associative array
      metric_data["$metric"]+="{ $labels_json, \"value\": $value },"
    fi
  done <<< "$metrics"

  # Write the metrics to the JSON file
  for key in "${!metric_data[@]}"; do
    echo "\"$key\": [ ${metric_data[$key]%?} ]," >> $METRICS_JSON_FILE
  done

  # Remove the last comma and close JSON structure
  sed -i '$ s/,$//' $METRICS_JSON_FILE
  echo "}" >> $METRICS_JSON_FILE
}

# Poll the API and convert metrics to JSON every 10 seconds in the background
polling() {
  while true; do
    poll_and_convert
    sleep 10
  done
}

# Start polling in the background
polling &

# Start the Python HTTP server to expose the JSON metrics
python3 <<EOF
import http.server
import socketserver
import json
import os

PORT = 8080
METRICS_FILE = "/tmp/metrics.json"

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/metrics":
            if os.path.exists(METRICS_FILE):
                with open(METRICS_FILE, "r") as file:
                    data = file.read()
                self.send_response(200)
                self.send_header("Content-type", "application/json")
                self.end_headers()
                self.wfile.write(data.encode())
            else:
                self.send_response(404)
                self.send_header("Content-type", "application/json")
                self.end_headers()
                self.wfile.write(json.dumps({"error": "Metrics file not found"}).encode())
        else:
            self.send_response(404)
            self.end_headers()

with socketserver.TCPServer(("", PORT), Handler) as httpd:
    print(f"Serving at port {PORT}")
    httpd.serve_forever()
EOF

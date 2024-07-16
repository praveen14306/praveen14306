#!/bin/bash

# Set the Kong Prometheus endpoint URL
API_URL="http://<kong-host>:<port>/metrics"

# File to store the JSON metrics
METRICS_JSON_FILE="/tmp/metrics.json"

# Function to poll the Kong endpoint and convert metrics to JSON
poll_and_convert() {
  # Poll the Kong Prometheus endpoint and get the plain text metrics
  metrics=$(curl -s $API_URL)

  # Convert the plain text metrics to JSON format
  json_metrics=$(echo "$metrics" | awk '
  BEGIN {
    print "{"
  }
  {
    if ($1 ~ /^[a-zA-Z_][a-zA-Z0-9_]*\{.*\}$/) {
      # Metric with labels
      metric = gensub(/([^{}]+)\{([^}]+)\} ([0-9.eE+-]+)/, "{\"name\": \"\\1\", \"labels\": {\\2}, \"value\": \\3}", "g", $0)
      print metric ","
    } else if ($1 ~ /^[a-zA-Z_][a-zA-Z0-9_]*$/) {
      # Metric without labels
      metric = gensub(/([^ ]+) ([0-9.eE+-]+)/, "{\"name\": \"\\1\", \"value\": \\2}", "g", $0)
      print metric ","
    }
  }
  END {
    print "}"
  }
  ' | sed '$s/,$//')  # Remove the last comma

  # Save the JSON metrics to a file
  echo "$json_metrics" > $METRICS_JSON_FILE
}

# Poll the API and convert metrics to JSON every 10 seconds
while true; do
  poll_and_convert
  sleep 10
done

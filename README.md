# SubD


\#!/bin/bash

rm subfinder.txt amass.txt hakrawler.txt gau.txt crt.txt final.txt final_output.txt

\# Check if the domain is provided as an argument
if [ -z "$1" ]; then
    echo "Usage: $0 <domain>"
    exit 1
fi

target_domain="$1"

\# File names
subfinder_file="subfinder.txt"
amass_file="amass.txt"
hakrawler_file="hakrawler.txt"
gau_file="gau_output"
gau_output="gau.txt"
crt_file="crt.txt"
final_file="final.txt"

\# Completion Percentage
total_steps=6

completed_steps=0
 
function increment_progress() {
    ((completed_steps++))
    percentage=$((completed_steps * 100 / total_steps))
    echo "Progress: $percentage%"
}

\# Step 1: Subfinder
subfinder -d "$target_domain" -all -o "$subfinder_file" > /dev/null 2>&1
increment_progress

\# Step 2: Amass 
timeout 300s amass enum -active -d "$target_domain" -o "$amass_file" > /dev/null 2>&1
increment_progress

\# Step 3: Hakrawler 
echo https://"$target_domain" | hakrawler |  grep -oP 'https?://\S+?\.com' | sed 's~https\?://~~' | sort -u > "$hakrawler_file" 
increment_progress

\# Step 4: Gau
timeout 20s gau "$target_domain" > "$gau_file"

\# Remove http:// or https:// prefixes from the file
grep -oP "https?://\K[^/]+" "$gau_file" | sort -u > "$gau_output" 
increment_progress

rm "$gau_file"

\# Step 5: crt.sh 
curl -s https://crt.sh/?q="$target_domain"\&output\=json | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > "$crt_file"
increment_progress

\# Merge subfinder, amass, and hakrawler output into a single file
cat "$subfinder_file" "$amass_file" "$crt_file" "$gau_output" "$hakrawler_file" | sort -u > "$final_file"

\# Step 6: httpx
httpxs -status-code -title -fc 404 -list "$final_file" > final_output.txt 
increment_progress

\# Display the final output
cat final_output.txt

### Crawl SERVICE.GOV.KH with bash script

Required tools: `htmlq`, `jq`, `awk`, `aria2` and `curl`

```shell
#!/usr/bin/env bash
set -o pipefail

# make a HEAD requests to many urls with many processors.
seq -w 101000 102000 | xargs -I{} -P $(nproc) \
	curl -I -w "%{http_code} %{url}\n" \
	-s -o /dev/null -H HEAD https://www.service.gov.kh/service/{} | tee service_links.txt

# filter the 200 status code only
awk '$1==200 {print $2}' service_links.txt > links.txt

# download html files with many processors at the same time.
aria2c -i links.txt -d html -j $(nproc)

echo "" > output.jsonl

for file in html/*; do
	title=$(htmlq --text title < $file)
	id=$(basename $file)
	ministry=$(htmlq --attribute href '#view_service > article > div.row > div.col-md-2.text-right.d-none.d-md-block > a' < $file)
	ministry_id=$(basename $ministry)	
	body=$(htmlq --text 'article .text-break' < $file)
	json="{\"id\": $id, \"title\": $(echo $title | jq -R), \"ministry_id\": $ministry_id, \"body\": $(echo $body | jq -Rs) }"
	echo $json >> output.jsonl
	echo $file
done
```
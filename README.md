### Crawl SERVICE.GOV.KH with bash script

```shell
#!/usr/bin/env bash
set -o pipefail

# make a HEAD requests to many urls with 16 processors.
seq -w 101000 102000 | xargs -I{} -P 16 \
	curl -I -w "%{http_code} %{url}\n" \
	-s -o /dev/null -H HEAD https://www.service.gov.kh/service/{} | tee service_links.txt

# filter the 200 status code only
awk '$1==200 {print $2}' service_links.txt > links.txt

# download html files with 16 processors at the same time.
aria2c -i links.txt -d html -j 16

echo "" > output.jsonl

for file in html/*; do
	title=$(htmlq --text title < $file)
	id=$(basename $file)
	ministry=$(htmlq --attribute href '#view_service > article > div.row > div.col-md-2.text-right.d-none.d-md-block > a' < $file)
	ministry_id=$(basename $ministry)	
	body=$(htmlq --text 'article .text-break' < $file)
	json="{\"id\": $id, \"title\": $(echo $title | jq -R), \"ministry_id\": $ministry_id, \"body\": $(echo $body | jq -R) }"
	echo $json >> output.jsonl
	echo $file
done

```
---
name: maven-check-deps
description: Check latest versions of Maven dependencies. Use when asked to check, verify, or find latest Maven dependency versions.
version: 1.0.0
allowed-tools: Bash
---

# Check latest Maven dependency versions

## Always use repo1.maven.org, never search.maven.org

`search.maven.org` Solr index lags behind real releases — always query
the authoritative source directly:

```bash
curl -s "https://repo1.maven.org/maven2/${GROUP//./\/}/$ARTIFACT/maven-metadata.xml" \
  | python3 -c "
import sys
from xml.etree import ElementTree as ET
root = ET.parse(sys.stdin).getroot()
print(root.find('versioning/release').text)
"
```

Use `versioning/release` (stable), not `versioning/latest` (may be a SNAPSHOT).

## Batch check example

```bash
for ga in "org.example:artifact" "com.other:lib"; do
  g=${ga%:*}; a=${ga#*:}
  latest=$(curl -s "https://repo1.maven.org/maven2/${g//.//}/$a/maven-metadata.xml" \
    | python3 -c "import sys; from xml.etree import ElementTree as ET; root=ET.parse(sys.stdin).getroot(); print(root.find('versioning/release').text)")
  echo "$ga → $latest"
done
```

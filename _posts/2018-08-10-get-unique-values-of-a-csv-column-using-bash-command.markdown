---
layout: post
title:  "Get Unique Values of a CSV Column Using Bash Command"
date:   2018-08-10 00:06:36 +0800
categories: bash
permalink: get-unique-values-of-a-csv-column-using-bash-command
---

Example input CSV file. The file is named <b>transaction.csv</b>

```csv
invoice_number,item_id,qty
#001,1,2
#001,2,2
#002,2,2
#003,1,2
#003,5,3
#004,2,2
#003,5,3
#005,2,2
```

The goal is to get the list of unique invoice_number

```
#001
#002
#003
#004
#005
```

## Step
1. Run awk command split by comma (-F), then print the first column. `awk -F ',' '{print $1}' transaction.csv`
2. Add sorting with unique params sort -u. Use -r to get reverse order
3. Final command is `awk -F ',' '{print $1}' transaction.csv | sort -u`

## Reference
- More detail about awk command [http://tldp.org/LDP/abs/html/awk.html](http://tldp.org/LDP/abs/html/awk.html)
- More detail about `sort -u` vs `sort | uniq` performance discussion at [stackoverflow thread](https://unix.stackexchange.com/questions/76049/what-is-the-difference-between-sort-u-and-sort-uniq)
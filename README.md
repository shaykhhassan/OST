# Open Source Tools And Scripting
## Assignment 2
#### Submitted By:
#### Sheikh Muhammad Hassan Ali - 23959698



# Empty_cell - Counter Script

This script given a text file version of a spreadsheet and the expected separator character, returns via standard output a list of the column titles (taken from the first line) and the number of empty cells found in that column.

### Step 1: Validate Script Usage

This part of script checks that exactly two arguments are provided. If more or less arguments are given it echo's a message for correct usage and exits.

```bash
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <input_file> <separator>" >&2
    exit 1
fi
```

### Step 2: Store Inputs Into Variables

The input arguments are stored in variables.

```bash
file="$1"
sep="$2"
```

### Step 3: Process the File Using `awk`

This part of the script uses `awk` with the field separator to process the input file line by line.

```awk
BEGIN {
    FS = sep
}
```

### Step 4: Handle the Header Row

For the first row (assuming it's always the header), this script:

- Stores each column name in an array `header`.
- Sets counters for empty cells per column which increments on each count of empty cell.
- Removes carriage return characters (`\r`) to handle Windows line endings.

```awk
NR == 1 {
    for (i = 1; i <= NF; i++) {
        gsub(/\r/, "", $i)
        header[i] = $i
        empty[i] = 0
    }
    max_fields = NF
    next
}
```

### Step 5: Count Empty Cells in Data Rows

For all other rows after header:

- Removes carriage return characters (`\r`) to handle Windows line endings.
- For each column up to `max_fields`, if a field is missing or empty, increment the empty cell counter.

```awk
{
    gsub(/\r/, "")
    for (i = 1; i <= max_fields; i++) {
        if (i > NF || $i == "") {
            empty[i]++
        }
    }
}
```

### Step 6: Output the Results

After processing all rows, the script prints each column's name and the number of empty cells in that column.

```awk
END {
    for (i = 1; i <= max_fields; i++) {
        print header[i] ": " empty[i]
    }
}
```

### Example Usage

```bash
./empty_cells bgg_dataset.txt ";"
```

Example output:

```bash
/ID: 16
Name: 0
Year Published: 1
Min Players: 0
Max Players: 0
Play Time: 0
Min Age: 0
Users Rated: 0
Rating Average: 0
BGG Rank: 0
Complexity Average: 0
Owned Users: 23
Mechanics: 1598
Domains: 10159
```

------

# Preprocess – Cleaning script

This script given a text file sends to standard output a cleaned version of the input file, where the following transformations have taken place:
- Convert the semicolon separator to the  <tab> character
- Convert the Microsoft line endings to Unix line endings
- Change format of floating-point numbers to use ‘.’ rather than ‘,’ as the decimal point
- Deal with non-ASCII characters by deleting them from the output
- For ID column searching the column for largest number, and then continue numbering past that if any field is empty, has 0 or invalid integer character
- Other empty cells are being replaced with 0

### Step 1: Validate Script Usage

This part of script checks that exactly one argument is provided. If more or less arguments are given it echo's a message for correct usage and exits.

```bash
if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <input_file>" >&2
    exit 1
fi
```

### Step 2: Store Input Into Variables

The input arguments are stored in variables.

```bash
input="$1"
```

### Step 3: Find the Highest Valid ID

Find the maximum valid numeric ID from the first column (excluding the header) to use for assigning new IDs where it has empty fields.

```bash
max_id=$(awk -F';' 'NR > 1 && $1 ~ /^[0-9]+$/ && $1 > max { max = $1 } END { print max+0 }' "$input")
next_id=$((max_id + 1))
```

### Step 4: Clean and Process the File Using `awk`

This part of the script uses `awk` to set field separator to semicolon, output field separator to tab.

```bash
awk -v OFS='\t' -v next_id="$next_id" '
BEGIN {
    FS = ";"
}
```

### Step 5: Clean All Fields

For each field in each row:

- Removes carriage return characters (`\r`) to handle Windows line endings.
- Remove non-ASCII characters such as `CO²` to `CO`.
- Replace commas with dots.
- Replace empty cells with zero (except header).

```awk
for (i = 1; i <= NF; i++) {
    gsub(/\r/, "", $i)
    gsub(/[^\x00-\x7F]/, "", $i)
    gsub(/,/, ".", $i)
    if (NR > 1 && $i == "") {
        $i = 0
    }
}
```

### Step 6: Print Header Row

Print the header row with fields separated by tabs.

```awk
if (NR == 1) {
    for (i = 1; i <= NF; i++) {
        printf "%s", $i
        if (i < NF) printf OFS
    }
    printf "\n"
    next
}
```

### Step 7: Assign or Fix IDs

For the first column (IDs), if missing, zero, or invalid, assign a new ID which is the next number in sequence to the highest number in ID column.

```awk
if ($1 == "" || $1 == 0 || $1 !~ /^[0-9]+$/) {
    $1 = next_id++
}
```

### Step 8: Skip Duplicate IDs

To improve cleaning process, this part does not print by skipping any duplicate rows based on duplicate ID's. Because ID is unique it should not be repeated.

```awk
if ($1 in seen) {
    next
}
seen[$1] = 1
```

### Step 9: Output Cleaned Rows

Print all fields separated by tabs.

```awk
for (i = 1; i <= NF; i++) {
    printf "%s", $i
    if (i < NF) printf OFS
}

printf "\n"
```

### Example Usage

```bash
./preprocess sample.txt
```
![Screenshot 2025-05-19 025221](https://github.com/user-attachments/assets/5ce25fca-e22b-4983-ae1a-77e326ba7bb4)

# Analysis - Script To Analyze Board Games

This script uses data from the cleaned input file to answer the four research questions listed below:

- What is the most popular domain and most popular mechanics across the set of games?

- What is the correlation between publication year and average rating?

- What is the correlation between game complexity and average rating?


### Step 1: Validate Script Usage

This part of script checks that exactly one argument is provided. If more or less arguments are given it echo's a message for correct usage and exits.

```bash
if [ "$#" -ne 1 ]; then
  echo "Usage: $0 <cleaned_input_file.tsv>" >&2
  exit 1
fi
```


### Step 2: Find Column Indices

This part of the script locates the index of all columns that need to be analyzed:

- Year Published
- Rating Average
- Complexity Average
- Mechanics
- Domains

```awk
NR == 1 {
  for (i = 1; i <= NF; i++) {
    if ($i == "Year Published") year_idx = i
    else if ($i == "Rating Average") rating_idx = i
    else if ($i == "Complexity Average") complexity_idx = i
    else if ($i == "Mechanics") mechanics_idx = i
    else if ($i == "Domains") domains_idx = i
  }
  next
}
```

### Step 3: Count Game Mechanics and Domains

This part of the script searched all rows. And splits the `Mechanics` and `Domains` fields by commas, trim spaces, and count total amount of each word in the column. It also make sures to only analyze fields with non-numeric values as `preprocess` replaces empty cells with '0' which might end up as the highest occurence.

```awk
NR>1 {
  # Count mechanics (split by comma)
  if ($mechanics_idx != "") {
    n = split($mechanics_idx, m_arr, ",")
    for(i=1; i<=n; i++) {
      gsub(/^ +| +$/, "", m_arr[i]) # trim spaces
      if(m_arr[i] ~ /[a-zA-Z]/) 
      mechanics_count[m_arr[i]]++
    }
  }

  # Count domains (split by comma)
  if ($domains_idx != "") {
    n = split($domains_idx, d_arr, ",")
    for(i=1; i<=n; i++) {
      gsub(/^ +| +$/, "", d_arr[i]) # trim spaces
      if(d_arr[i] ~ /[a-zA-Z]/)
      domain_count[d_arr[i]]++
    }
  }
```

### Step 4: Collect Data for Correlation

Store numeric values for year published, rating average, and complexity average to calculate correlations.

```awk
# Store values for correlation if numeric and not empty
  year = $year_idx + 0
  rating = $rating_idx + 0
  complexity = $complexity_idx + 0
```

### Step 5: Determine Most Popular Mechanics and Domains

Identify the mechanic and domain with the highest frequency.

```awk
END {
  # Find most popular mechanic
  max_mech_count = 0
  max_mech = ""
  for(m in mechanics_count) {
    if(mechanics_count[m] > max_mech_count) {
      max_mech_count = mechanics_count[m]
      max_mech = m
    }
  }

  # Find most popular domain
  max_dom_count = 0
  max_dom = ""
  for(d in domain_count) {
    if(domain_count[d] > max_dom_count) {
      max_dom_count = domain_count[d]
      max_dom = d
    }
  }
```
### Step 6: Compute Pearson Correlation Coefficients

Calculate correlations between:

- Year Published and Rating Average
- Complexity Average and Rating Average
#### Pearson Correlation Coefficients
$$
r = \frac{n \sum (XY) - \sum X \sum Y}{\sqrt{[n \sum X^2 - (\sum X)^2][n \sum Y^2 - (\sum Y)^2]}}
$$

```awk
# Calculate Pearson correlations
  sum_x = sum_y = sum_x2 = sum_y2 = sum_xy = 0

  # Year vs Rating
  for(i in years) {
    x = years[i]
    y = ratings[i]
    sum_x += x
    sum_y += y
    sum_x2 += x*x
    sum_y2 += y*y
    sum_xy += x*y
  }

  n = count
  numerator = sum_xy - (sum_x*sum_y)/n
  denominator = sqrt( (sum_x2 - (sum_x*sum_x)/n) * (sum_y2 - (sum_y*sum_y)/n) )
  corr_year_rating = (denominator != 0) ? numerator/denominator : 0

  # Complexity vs Rating
  sum_x = sum_y = sum_x2 = sum_y2 = sum_xy = 0
  for(i in complexities) {
    x = complexities[i]
    y = ratings[i]
    sum_x += x
    sum_y += y
    sum_x2 += x*x
    sum_y2 += y*y
    sum_xy += x*y
  }

  numerator = sum_xy - (sum_x*sum_y)/n
  denominator = sqrt( (sum_x2 - (sum_x*sum_x)/n) * (sum_y2 - (sum_y*sum_y)/n) )
  corr_complexity_rating = (denominator != 0) ? numerator/denominator : 0
```
### Example Usage

```bash
./analysis bgg_dataset.tsv
```

Example output:

```bash
The most popular game mechanics is Hand Management found in 432 games
The most popular game domain is Wargames found in 3029 games
The correlation between the year of publication and the average rating is 0.136
The correlation between the complexity of a game and its average rating is 0.511
```

#### Script Authored By: Sh M Hassan Ali (23959698) UWA MIT
#### Assignment 2 - Open Source Tools and Scripting

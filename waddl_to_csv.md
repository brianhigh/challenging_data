---
title: "WADDL to CSV"
author: "Brian High"
date: "2023-04-24"
output:
  html_document:
    keep_md: yes
editor_options: 
  chunk_output_type: console
---



## Read WADDL file and convert to CSV

We will use an example "text" data file, a modified subset of a file we were told 
came from the [Washington Animal Disease Diagnostic Laboratory](https://waddl.vetmed.wsu.edu/) (WADDL). We were asked to read the file into R, but we were not told anything 
about the file format or structure, except that it was a "text" file.

This particular file format is challenging because the functions you would normally 
use for reading "text" files will not work without using some special options. 
Further, the data are not arranged in a familiar and [tidy](https://tidyr.tidyverse.org/articles/tidy-data.html) "columns are variables" structure.

We will explore how to address the issues that come up when reading this file: 

- Unexpected "embedded nulls" warning
- Unexpected "invalid multibyte string" error
- Unexpected file encoding type
- Unexpected delimiter (tab)
- Missing "elements" (values)
- Missing header
- Columns with no values (to remove)
- Rows missing any useful data (to remove)
- Unusual data structure (not "columns are variables")

For the curious, we also provide more details in three appendices:

- Appendix I: What's an "embedded null"?
- Appendix II: What's UTF-16LE?
- Appendix III: Good nulls and bad nulls

## Setup

First, we will load our R packages, set options, and set file paths.


```r
# Attach packages, installing as needed
if (!requireNamespace("pacman", quietly = TRUE)) install.packages("pacman")
pacman::p_load(formatR, knitr, readr, dplyr, tidyr, stringr, purrr)

# Set options
options(readr.show_col_types = FALSE, readr.show_progress = FALSE)

# Set file paths
txt_file <- file.path("data", "waddl_data_example.txt")
csv_file <- file.path("data", "waddl_data_example.csv")
```

## Data file inspection

The data file we were given had "Sensititre" in the name and ended with a ".TXT" 
suffix. An internet search revealed that [Sensititre](https://www.thermofisher.com/order/catalog/product/V3000-VZ) is a laboratory instrument 
used for antimicrobial sensitivity testing. We could not find a description of the 
file format and we were not provided with a code book to explain the variables. 
There is commercial software available for purchase that was designed to
work with this instrument called the Thermo Scientific™ Sensititre™ SWIN™ Software 
System. We do not have this software, nor could we find a manual for it available 
online.

The file can be opened in Notepad and [Notepad++](https://notepad-plus-plus.org/), 
but will not show much when opened in R's text editor. In Notepad++ we turned on 
View -> Show Symbol -> Show All Characters. We see it has no column headers, 
is tab delimited, contains NUL characters, and the columns all line up like a 
fixed-width format, and the lines end with CRLF, or "carriage-return line-feed", the 
standard text file line ending characters used on Windows systems). In Notepad++, 
from the Encoding menu, we see "UTF-16 LE BOM". 

Opening the file on a Mac with [BBEdit](https://www.barebones.com/products/bbedit/), 
we selected View -> Text Display -> Show Invisibles. This showed the tabs as grey 
triangles and there was an upside-down red question mark where there were no data 
values. Maybe that's for the NULs. At the bottom in the status bar, it says 
"Text File", "Unicode (UTF-16 Little-Endian)" and "Windows (CRLF)". So, that's 
the same file encoding as Notepad++ reported, but without the "BOM". If you 
select the file encoding in the status bar of BBEdit, you see that there is 
another choice for "Unicode (UTF-16 Little-Endian, no BOM)", so that means 
the one selected for us must mean "with BOM" implicitly.

So, we expect a normal "text" file, because of the file suffix, but with NUL 
characters and an atypical file encoding. Normally, we would expect UTF-8, ANSI 
(Windows-1252 or CP-1252), ASCII, or Latin-1 (ISO/IEC 8859-1), but it's strange 
the file does not show more than the first character of the file ("1"). Maybe 
the encoding type or NUL characters are not supported by R's text editor.  

We could convert the file format in either Notepad++ or BBedit to UTF-8 and make 
our lives a little easier, but we will try to do all processing in R for 
better automation and reproducibility.

## Embedded nulls

We will start with `read.table()`, specifying a tab delimiter and no header, 
and reading in hust the first line. When we read it in we get some unexpected 
messages.


```r
# Try reading with read.table, just the first row
try(read.table(txt_file, sep = "\t", header = FALSE, nrows = 1))
```

```
## Warning in read.table(txt_file, sep = "\t", header = FALSE, nrows = 1): line 1
## appears to contain embedded nulls
```

```
## Error in type.convert.default(data[[i]], as.is = as.is[i], dec = dec,  : 
##   invalid multibyte string at '<ff><fe>1'
```

We see a warning "appears to contain embedded nulls". We saw NUL characters in 
Notepad++, and red upside-down question marks in BBEdit for empty fields, so 
perhaps that's what this is about. The warning will go away if we use `skipNul = TRUE`.


```r
# Try reading with read.table, just the first row
try(read.table(txt_file, sep = "\t", header = FALSE, nrows = 1, skipNul = TRUE))
```

```
## Error in type.convert.default(data[[i]], as.is = as.is[i], dec = dec,  : 
##   invalid multibyte string at '<ff><fe>1'
```

We still see the "invalid multibyte string" error.

## Invalid multibyte strings

The "invalid multibyte string" error is a clue that the file is not encoded as 
expected (usually UTF-8 or ASCII), so we check to see what it might be instead.


```r
# Check file encoding
readr::guess_encoding(txt_file, n_max = 1)
```

```
## # A tibble: 4 × 2
##   encoding   confidence
##   <chr>           <dbl>
## 1 UTF-16LE         1   
## 2 ISO-8859-1       0.68
## 3 ISO-8859-2       0.54
## 4 UTF-32LE         0.25
```

And the guesses are listed with the most likely encoding, "UTF-16LE", at the the 
top of the list with a probability of 1. Notepad++ showed the same encoding, 
but added "BOM". We can verify that encoding by checking the first four bytes 
of the file for the "byte order mark" (BOM). For UTF-16, we would expect a 
two-byte BOM, so checked the next two bytes after that will also let us see the 
first two-byte character of data.


```r
# Look for a byte order mark (BOM) in the first four bytes to verify encoding
readBin(txt_file, what = "raw", n = 4)
```

```
## [1] ff fe 31 00
```

We see a two-byte BOM of `ff fe`, followed by the first two-byte character of 
our data, "1", and "1" is `31 00` in UTF-16LE encoding. This verifies UTF-16LE 
BOM encoding.

- See: https://www.unicode.org/faq/utf_bom.html#bom4
- And: https://en.wikipedia.org/wiki/Byte_order_mark#UTF-16

The BOM is the source of the "invalid multibyte string" error because `read.table()` 
does not know this is a UTF-16LE file having the associated BOM. 

The `\xff` and `\xfe` are the hexadecimal values for these two special "bytes", 
where a byte is 16 "bits". 

When we set `fileEncoding = "UTF-16LE"`, reading the first line, there are no 
errors. The BOM was removed automatically.


```r
read.table(txt_file, sep = "\t", header = FALSE, nrows = 1, skipNul = TRUE, fileEncoding = "UTF-16LE")
```

```
##   V1    V2        V3 V4 V5 V6          V7 V8 V9   V10 V11 V12 V13 V14 V15 V16
## 1  1 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
##   V17 V18 V19 V20 V21 V22 V23 V24 V25 V26 V27 V28 V29 V30 V31 V32 V33 V34 V35
## 1  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V36 V37 V38 V39 V40                 V41 V42 V43 V44 V45 V46 V47 V48 V49 V50
## 1  NA  NA  NA  NA  24 2099-05-25 13:27:05  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V51 V52 V53 V54 V55 V56 V57 V58 V59 V60 V61 V62 V63 V64 V65 V66 V67 V68 V69
## 1  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V70 V71 V72 V73 V74 V75 V76 V77 V78 V79 V80 V81 V82 V83 V84 V85 V86 V87 V88
## 1  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V89 V90 V91 V92 V93 V94 V95 V96 V97 V98 V99 V100 V101 V102 V103 V104 V105
## 1  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA   NA   NA   NA   NA   NA   NA
##   V106 V107 V108 V109 V110 V111 V112 V113 V114 V115 V116 V117 V118 V119 V120
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V121 V122 V123 V124 V125 V126 V127 V128 V129 V130 V131 V132 V133 V134 V135
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V136 V137 V138 V139 V140 V141 V142 V143 V144 V145 V146 V147 V148 V149 V150
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V151 V152 V153 V154 V155 V156 V157 V158 V159 V160 V161 V162 V163 V164 V165
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V166 V167 V168 V169 V170 V171 V172 V173 V174 V175 V176 V177 V178 V179 V180
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V181 V182 V183 V184 V185 V186 V187 V188 V189 V190 V191 V192 V193 V194 V195
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V196 V197 V198 V199 V200 V201 V202 V203 V204 V205 V206 V207 V208 V209 V210
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V211 V212 V213 V214 V215 V216 V217 V218 V219 V220 V221 V222 V223 V224 V225
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V226 V227 V228 V229 V230 V231 V232 V233 V234 V235 V236 V237 V238 V239 V240
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V241 V242 V243 V244 V245 V246 V247 V248 V249 V250 V251 V252 V253 V254 V255
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V256 V257 V258 V259 V260 V261 V262 V263 V264 V265 V266 V267 V268 V269 V270
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V271 V272 V273 V274 V275 V276 V277 V278 V279 V280 V281 V282 V283 V284 V285
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V286 V287 V288 V289 V290 V291 V292 V293 V294 V295 V296 V297 V298 V299 V300
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V301 V302 V303 V304 V305 V306 V307 V308 V309 V310 V311 V312 V313 V314 V315
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V316 V317 V318 V319 V320 V321 V322 V323 V324 V325 V326 V327 V328 V329 V330
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
##   V331 V332 V333 V334 V335 V336 V337 V338 V339 V340 V341
## 1   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA   NA
```

## Read more rows

We get no errors when we try to read all of the rows.


```r
# Read file
df <- read.table(txt_file, sep = "\t", header = FALSE, skipNul = TRUE, fileEncoding = "UTF-16LE")

# View data
df[1:5, 1:50]
```

```
##   V1    V2        V3 V4 V5 V6          V7 V8 V9   V10 V11 V12 V13 V14 V15 V16
## 1  1 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 2  2 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 3  3 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 4  4 99057 2099-0031 NA NA NA 2099-0031.1  1 NA E.COL ANY EQU  NA  -1  NA  NA
## 5  5 99058 2099-0129 NA NA NA 2099-0129.1  1 NA E.COL   U CAN  NA  -1  NA  NA
##   V17 V18 V19 V20 V21 V22 V23 V24 V25 V26 V27 V28 V29 V30 V31 V32 V33 V34 V35
## 1  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 2  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 3  NA  NA   0   0  NA  NA   +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 4  NA  NA   0   0  NA  NA      NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 5  NA  NA   0   0  NA  NA      NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V36 V37 V38 V39 V40                 V41    V42        V43    V44    V45
## 1  NA  NA  NA  NA  24 2099-05-25 13:27:05                                
## 2  NA  NA  NA  NA  24 2099-06-01 09:14:28                                
## 3  NA  NA  NA  NA  24 2099-06-01 13:42:48 AMIKAC <=      16 NOINTP AMOCLA
## 4  NA  NA  NA  NA  18 2099-06-02 10:24:16 AMIKAC <=       4   SUSC       
## 5  NA  NA  NA  NA  24 2099-06-05 08:22:22 AMIKAC <=       4   SUSC AMOCLA
##          V46   V47    V48        V49    V50
## 1                                          
## 2                                          
## 3  =     0.5 INTER AMPICI  =       4 RESIST
## 4                  AMPICI  >      32 RESIST
## 5  =       4  SUSC AMPICI  =       4   SUSC
```

But when we view the data, we see there are some blank fields in columns V23, V42, 
and columns after that. We would prefer to see NAs here. Let's see what's really 
in those empty fields.


```r
unique(df$V23)
```

```
## [1] "+" ""  "-"
```

```r
unique(df$V42)
```

```
## [1] ""       "AMIKAC"
```

So, the blank fields are really the empty string (""). We will read in the 
file again setting `na.strings = ""`.


```r
# Read file
df <- read.table(txt_file, sep = "\t", header = FALSE, skipNul = TRUE, fileEncoding = "UTF-16LE", na.strings = "")

# View data
df[1:5, 1:50]
```

```
##   V1    V2        V3 V4 V5 V6          V7 V8 V9   V10 V11 V12 V13 V14 V15 V16
## 1  1 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 2  2 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 3  3 99050 2099-0707 NA NA NA 2099-0707.2  2 NA S.AUR ANY ANY  NA  -1  NA  NA
## 4  4 99057 2099-0031 NA NA NA 2099-0031.1  1 NA E.COL ANY EQU  NA  -1  NA  NA
## 5  5 99058 2099-0129 NA NA NA 2099-0129.1  1 NA E.COL   U CAN  NA  -1  NA  NA
##   V17 V18 V19 V20 V21 V22  V23 V24 V25 V26 V27 V28 V29 V30 V31 V32 V33 V34 V35
## 1  NA  NA   0   0  NA  NA    +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 2  NA  NA   0   0  NA  NA    +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 3  NA  NA   0   0  NA  NA    +  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 4  NA  NA   0   0  NA  NA <NA>  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
## 5  NA  NA   0   0  NA  NA <NA>  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA  NA
##   V36 V37 V38 V39 V40                 V41    V42        V43    V44    V45
## 1  NA  NA  NA  NA  24 2099-05-25 13:27:05   <NA>       <NA>   <NA>   <NA>
## 2  NA  NA  NA  NA  24 2099-06-01 09:14:28   <NA>       <NA>   <NA>   <NA>
## 3  NA  NA  NA  NA  24 2099-06-01 13:42:48 AMIKAC <=      16 NOINTP AMOCLA
## 4  NA  NA  NA  NA  18 2099-06-02 10:24:16 AMIKAC <=       4   SUSC   <NA>
## 5  NA  NA  NA  NA  24 2099-06-05 08:22:22 AMIKAC <=       4   SUSC AMOCLA
##          V46   V47    V48        V49    V50
## 1       <NA>  <NA>   <NA>       <NA>   <NA>
## 2       <NA>  <NA>   <NA>       <NA>   <NA>
## 3  =     0.5 INTER AMPICI  =       4 RESIST
## 4       <NA>  <NA> AMPICI  >      32 RESIST
## 5  =       4  SUSC AMPICI  =       4   SUSC
```

So, now the blank fields have NA instead.

## Using `readr::read_delim`

We have been using `base`-R functions to read the file. We could use `readr` 
functions instead, but they do not handle the embedded nulls. So, you could 
work around that by processing with `base`-R's `readLines`  and `writeLines` 
first.


```r
# Do the same with readr::read_delim(), but after using readLines/writeLines since read_delim can't
# handle the embedded nulls
temp <- tempfile()
readLines(txt_file, encoding = "UTF-16LE", skipNul = TRUE) %>%
    writeLines(temp)
df <- read_delim(temp, delim = "\t", col_names = FALSE, na = "", name_repair = make.names)
res <- file.remove(temp)
```

We might also consider using `fread` from the `data.table` package, but it 
doesn't support UTF-16 encoding. So, again, we would be forced to read the 
file with something else first. It's hard to justify the extra trouble.

## Check dimensions


```r
# Check dimensions
dim(df)
```

```
## [1]  20 341
```

That's a lot of columns! Maybe more than we need...

# Remove columns with no data

When we looked at the lines, we saw lots of empty values, so let's try to 
remove columns that contain no data, as they are just taking up space.


```r
# Remove columns containing only NA values
df <- df %>%
    select(where(~sum(is.na(.x)) < length(.x)))

# Check dimensions
dim(df)
```

```
## [1]  20 146
```

```r
# View data
df[1:5, 1:20]
```

```
##   V1    V2        V3          V7 V8   V10 V11 V12 V14 V19 V20  V23 V40
## 1  1 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY  -1   0   0    +  24
## 2  2 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY  -1   0   0    +  24
## 3  3 99050 2099-0707 2099-0707.2  2 S.AUR ANY ANY  -1   0   0    +  24
## 4  4 99057 2099-0031 2099-0031.1  1 E.COL ANY EQU  -1   0   0 <NA>  18
## 5  5 99058 2099-0129 2099-0129.1  1 E.COL   U CAN  -1   0   0 <NA>  24
##                   V41    V42        V43    V44    V45        V46   V47
## 1 2099-05-25 13:27:05   <NA>       <NA>   <NA>   <NA>       <NA>  <NA>
## 2 2099-06-01 09:14:28   <NA>       <NA>   <NA>   <NA>       <NA>  <NA>
## 3 2099-06-01 13:42:48 AMIKAC <=      16 NOINTP AMOCLA  =     0.5 INTER
## 4 2099-06-02 10:24:16 AMIKAC <=       4   SUSC   <NA>       <NA>  <NA>
## 5 2099-06-05 08:22:22 AMIKAC <=       4   SUSC AMOCLA  =       4  SUSC
```

## Remove rows with no drug data

By inspection, we see the drug data (name, quantity, etc.) starts at column V42. 
Some rows do not have any data from that column onwards, so we will remove those 
rows, as we don't need them.


```r
# Find the numerical index of column V42 to use for subsetting
drug_start <- grep("V42", names(df))
drug_start
```

```
## [1] 15
```

```r
# Remove rows containing no drug data, e.g., where columns V42 onward are all NA
df <- df %>%
    filter(!if_all(all_of(drug_start:ncol(df)), ~is.na(.)))

# Check dimensions
dim(df)
```

```
## [1]  18 146
```

## Examine structure of drug data

When we look at the columns of drug data, we see a strange structure.


```r
# View data
df %>%
    select(V42:V50) %>%
    head()
```

```
##      V42        V43    V44    V45        V46    V47    V48        V49    V50
## 1 AMIKAC <=      16 NOINTP AMOCLA  =     0.5  INTER AMPICI  =       4 RESIST
## 2 AMIKAC <=       4   SUSC   <NA>       <NA>   <NA> AMPICI  >      32 RESIST
## 3 AMIKAC <=       4   SUSC AMOCLA  =       4   SUSC AMPICI  =       4   SUSC
## 4 AMIKAC <=       4   SUSC AMOCLA  =       2   SUSC AMPICI  >       8 RESIST
## 5 AMIKAC  >      32 RESIST AMOCLA  >       8 RESIST AMPICI  >       8 RESIST
## 6 AMIKAC  >      32 RESIST AMOCLA  >       8 RESIST AMPICI  >       8 RESIST
```

By inspection, we see columns containing drug data are in sets of 3:
drug_name drug_quantity drug_qualifier, example: "AMIKAC" " <= 16" "NOINTP".

## Reshape drug columns

We would prefer a structure where there were two colums per drug: one for the 
quantity and the other for the qualifier, which we will call "RIS". We want 
the two colums to be named after the drug, which is contained in the first 
of the three columns for each set. So, we will extract these first.


```r
# Identify columns containing only a single abbreviated drug name (or NA)
drug_cols <- names(df)[drug_start:ncol(df)][drug_start:ncol(df)%%3 == 0]
col_vals <- map(drug_cols, ~table(df[, .x])) %>%
    setNames(drug_cols)

# Confirm all of these variables only contain 1 value matching name pattern
table(unlist(map(col_vals, length)))
```

```
## 
##  1 
## 44
```

```r
table(unlist(map(col_vals, ~grepl("^[A-Z]+$", names(.x)[1]))))
```

```
## 
## TRUE 
##   44
```

```r
# Extract the drug names and column number
drugs <- sapply(col_vals, names)
drug_nums <- as.numeric(gsub("^V", "", drug_cols))

# Rename drug columns in sets of 3 columns for each drug: name, value, and RIS
drugs_df <- tibble(drug = drugs, drug_num = drug_nums) %>%
    mutate(col1_old = paste0("V", drug_nums), col2_old = paste0("V", drug_nums + 1), col3_old = paste0("V",
        drug_nums + 2)) %>%
    mutate(col1_new = paste0(drug, "_NAME"), col2_new = paste0(drug, "_VAL"), col3_new = paste0(drug,
        "_RIS"))
lookup <- with(drugs_df, c(col1_old, col2_old, col3_old) %>%
    setNames(c(col1_new, col2_new, col3_new)))
df <- rename(df, any_of(lookup))
```

## Final cleanup

Now, all we need to do is remove extra spaces from the value columns, abbreviate 
the names of the drug value columns, and remove the original drug name columns. 
We won't worry about providing column names for the non-drug columns right now.


```r
# Clean up data
df <- df %>%
    mutate(across(ends_with("_VAL"), ~str_replace_all(.x, "\\s+", " "))) %>%
    rename_with(~str_remove(., "_VAL")) %>%
    select(-ends_with("_NAME"))

# View results
df[1:5, 15:20]
```

```
##   AMIKAC AMIKAC_RIS AMOCLA AMOCLA_RIS AMPICI AMPICI_RIS
## 1  <= 16     NOINTP  = 0.5      INTER    = 4     RESIST
## 2   <= 4       SUSC   <NA>       <NA>   > 32     RESIST
## 3   <= 4       SUSC    = 4       SUSC    = 4       SUSC
## 4   <= 4       SUSC    = 2       SUSC    > 8     RESIST
## 5   > 32     RESIST    > 8     RESIST    > 8     RESIST
```

After saving the results to a CSV file, we are done.


```r
# Save result as CSV
write_csv(df, csv_file)
```

## Appendix I: What's an "embedded null"?

A "embedded null" is a non-printing character (NUL) that is sometimes used to 
terminate a character string. You can see them using `readBin()`. This will 
show the hexadecimal equivalents of the characters. The nulls are the 00 values.


```r
x <- readBin(txt_file, what = "raw", n = 16)
x
```

```
##  [1] ff fe 31 00 09 00 39 00 39 00 30 00 35 00 30 00
```

And we can "decode" the hexadecimal characters with `rawToChar()`, if we remove
the nulls first:


```r
paste(rawToChar(x[x != 0], multiple = TRUE), collapse = " ")
```

```
## [1] "\xff \xfe 1 \t 9 9 0 5 0"
```

## Appendix II: What's UTF-16LE?

UTF-16 is a file encoding for 16-bit (2-byte) Unicode characters. We suspected we 
had an UTF-16LE file because of the two special bytes starting the file. These 
particular bytes, `FF FE` specify that the encoding is "little endian" (LE), giving us 
UTF-16LE encoding. 

- See: https://en.wikipedia.org/wiki/Endianness
- And: https://www.tutorialspoint.com/big-endian-and-little-endian

If we had a UTF-32LE file, it would have started with `FF FE 00 00`. If the file 
was UTF-8, it would either not have a BOM or the BOM would be `EF BB BF`. 

Because UTF-8 is represented with single bytes, there is no big or little "end". 
Endianness is to specify byte order, so you need more than one byte per character to 
have a byte order. So, the difference between "LE" and "BE" are just the order 
of the two bytes in each multibyte character. We can see the difference with 
an example that compares the bytes of LE and BE representing the character "1":


```r
x <- "1"
Encoding(x) <- "UTF-8"
x8 <- charToRaw(x)
x16LE <- iconv(x, "UTF-8", "UTF-16LE", toRaw = TRUE)
x16BE <- iconv(x, "UTF-8", "UTF-16BE", toRaw = TRUE)
kable(t(c(x = x, `UTF-8` = x8, `UTF-16LE` = x16LE, `UTF-16BE` = x16BE)))
```



|x  |UTF-8        |UTF-16LE              |UTF-16BE              |
|:--|:------------|:---------------------|:---------------------|
|1  |as.raw(0x31) |as.raw(c(0x31, 0x00)) |as.raw(c(0x00, 0x31)) |

Those `00` values look like the dreaded "embedded nulls", right? But now we see 
those are valid after all. They are simply one of the two bytes of some UTF-16 
characters. That's why the warning said, *appears* to contain embedded nulls. 

## Appendix III: Good nulls and bad nulls

It turns out we actually need to "skip" the embedded nulls (`0x00 0x00`), or we 
will just read a few values.


```r
read.table(txt_file, nrows = 1, fileEncoding = "UTF-16LE") %>%
    suppressWarnings()
```

```
##   V1    V2        V3
## 1  1 99050 2099-0707
```

The reason is that we actually do have real embedded nulls after the third "cell".


```r
readBin(txt_file, what = "raw", n = 100)
```

```
##   [1] ff fe 31 00 09 00 39 00 39 00 30 00 35 00 30 00 09 00 32 00 30 00 39 00 39
##  [26] 00 2d 00 30 00 37 00 30 00 37 00 09 00 00 00 09 00 00 00 09 00 00 00 09 00
##  [51] 32 00 30 00 39 00 39 00 2d 00 30 00 37 00 30 00 37 00 2e 00 32 00 09 00 32
##  [76] 00 09 00 00 00 09 00 53 00 2e 00 41 00 55 00 52 00 09 00 41 00 4e 00 59 00
```

You can see that between some tabs (`\t` or `09 00`) we have `00 00` values. These
are the real embedded nulls because there are two of them together, so 16 bits of 
zeros.

I'm guessing these are the actual `NA` values of this file format. Since we don't 
have column headings, or any code book at all, we can't be sure, but if the tab 
is actually the delimiter, then this is how the empty "cells" are presented in 
this file. So, maybe we can try setting the null as a NA value:


```r
try(read.table(txt_file, nrows = 1, fileEncoding = "UTF-16LE", na.strings = "\x00"))
```

```
Error: nul character not allowed (line 1)
```

Unfortunately we are not allowed to enter nulls here. So, we just skip them. 
That will leave the empty cells truly empty ("") and so we use the empty 
string as the NA string with `na.strings = ""`. 

At least now we have a better idea of what embedded nulls are, why 
they're there (probably), and why it's (probably) okay to skip them, at least in 
this situation.

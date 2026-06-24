sort
sort –urn access.log
Flag	Meaning
-u	Unique lines only
-r	Reverse order
-n	Numeric sort

grep
grep –roE ‘pattern’ ./logs
grep –wrE '[0-9]{2}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}'  ./log-test
Flag	Meaning
-v	Invert match
-r	Recursive search
-n	Show line numbers
-c	Count matches
-o	Print matched text only
-E	Extended regex
-w	Whole word match
-h        Hide filename prefix

find
find log-test/ -type f -name ‘specific name’
Flag	Meaning
-type f	Files only
-type d	Directories only
-name	Filename pattern

xargs – create line arguments

uniq
sort file | uniq –cd  | syslog.log
Flag	Meaning
-c	Count occurrences
-d	Show duplicates only
-u	Show unique lines only

cut
cut –d’:’ –f1 syslog.log
Flag	Meaning
-d X	Delimiter
-f N	Field number

wc
wc –lw syslog.log
Flag	Meaning
-l	counts Lines
-w	counts Words

du
du –h file/directory
Flag	Meaning
-s	Summary only
-h	Human-readable

df
df -h
-h       Disk storage

ls
ls –lah
Flag	Meaning
-l	Long listing
-h	Human-readable sizes
-a	Include hidden files

tee
tee –a output.txt
Flag	Meaning
-a	Append instead of overwrite

tr
echo “23.53.23.42” | tr -d ’.‘ 
Flag	Meaning
-d	Delete characters
-s	Squeeze repeated characters into one
‘a-z’ ‘A-Z’

Quick Reference Card (students may keep this page)
Allowed tools: grep cut find xargs du df ls sort wc uniq tr tee 
Allowed shell: pipes |, redirects > >> <, wildcards * ? 
Not allowed: awk sed if for while \ (line continuation) — keep commands on one line

# Unique sorted lines
sort -u file.log

# Count non-empty lines
grep -vc '^[[:space:]]*$' file.log

# Extract all IPv4 addresses
grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' file.log | sort -u

# Search all files in a directory recursively (no filename prefix)
find logs/ -type f | xargs grep -h "pattern"
alternative: grep –rh “pattern” <directory or file>

# Count how many times each line appears
sort file.log | uniq -c | sort -rn

# Filter lines NOT matching a pattern
grep -v "pattern" file.log

# Print specific field (e.g. field 2, space-delimited)
cut -d' ' -f2 file.log

# Write output AND still see it on screen
grep "pattern" file.log | tee output.txt

# Append to file
grep "pattern" file.log >> output.txt


# Count words/lines/characters
wc -l file.log
wc -w file.log

# Disk usage of log directory
du -sh logs/

# List files with sizes
ls -lh logs/auth/


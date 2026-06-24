Quick Reference Card (students may keep this page) 
Allowed tools: grep cut find xargs du df ls sort wc uniq tr tee  
Allowed shell: pipes |, redirects > >> <, wildcards * ?  
Not allowed: awk sed if for while \ (line continuation) — keep commands on one 
line 
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

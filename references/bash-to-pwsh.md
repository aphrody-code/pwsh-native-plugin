# Bash → PowerShell pattern reference

Canonical cheat sheet for translating Bash idioms to PowerShell. Compiled from [TheShellNut](https://mathieubuisson.github.io/powershell-linux-bash/), [Ironman Software](https://blog.ironmansoftware.com/daily-powershell/bash-powershell-cheatsheet), and [Secure Ideas](https://www.secureideas.com/blog/from-linux-to-powershell-and-back-a-quick-command-reference).

**Core mindset shift:** Bash pipes bytes/text. PowerShell pipes **objects** with typed properties and methods. Don't re-implement a grep pipeline in PS if you can filter a property directly.

---

## 1. Filesystem navigation

| Bash | PowerShell | Notes |
| --- | --- | --- |
| `pwd` | `Get-Location` / `$PWD` | Alias: `gl`. Returns a `PathInfo` object. |
| `cd <path>` | `Set-Location <path>` | Aliases: `cd`, `sl`, `chdir`. `cd -` → `Set-Location -` (needs PSReadLine). |
| `cd ~` | `Set-Location ~` / `Set-Location $HOME` | |
| `pushd` / `popd` | `Push-Location` / `Pop-Location` | Aliases: `pushd`, `popd`. |
| `ls` | `Get-ChildItem` | Aliases: `ls`, `dir`, `gci`. Returns `FileInfo` / `DirectoryInfo`. |
| `ls -la` | `Get-ChildItem -Force` | `-Force` shows hidden + system files. |
| `ls -lh` | `Get-ChildItem \| Format-Table -AutoSize` | |
| `ls -ltr` | `Get-ChildItem \| Sort-Object LastWriteTime` | Object sort, not text parse. |
| `ls *.txt` | `Get-ChildItem -Filter *.txt` | `-Filter` is faster than `-Include` (uses FS filter). |
| `ls -R` | `Get-ChildItem -Recurse` | |
| `find . -name '*.log'` | `Get-ChildItem -Recurse -Filter *.log` | |
| `find . -type f -size +10M` | `Get-ChildItem -Recurse -File \| Where-Object Length -gt 10MB` | `10MB` is a built-in literal. |
| `find . -mtime -7` | `Get-ChildItem -Recurse \| Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-7) }` | |

## 2. File I/O

| Bash | PowerShell | Notes |
| --- | --- | --- |
| `cat file` | `Get-Content file` | Aliases: `cat`, `gc`, `type`. Emits one string per line. |
| `cat file \| head -n 20` | `Get-Content file -TotalCount 20` | Or `\| Select-Object -First 20`. |
| `tail -n 50 file` | `Get-Content file -Tail 50` | |
| `tail -f file` | `Get-Content file -Wait` | |
| `echo 'x' > file` | `'x' \| Set-Content file` | Or `'x' > file`. `Set-Content` picks encoding from `-Encoding`. |
| `echo 'x' >> file` | `'x' \| Add-Content file` | Or `'x' >> file`. |
| `touch file` | `New-Item file -ItemType File` | Or `$null > file` / `(Get-Item file).LastWriteTime = Get-Date`. |
| `mkdir -p a/b/c` | `New-Item a/b/c -ItemType Directory -Force` | `-Force` creates intermediate dirs and is idempotent. |
| `cp src dst` | `Copy-Item src dst` | Alias: `cp`, `cpi`, `copy`. |
| `cp -r src dst` | `Copy-Item src dst -Recurse` | |
| `mv src dst` | `Move-Item src dst` | Alias: `mv`, `mi`, `move`. |
| `rm file` | `Remove-Item file` | Aliases: `rm`, `ri`, `del`. |
| `rm -rf dir` | `Remove-Item dir -Recurse -Force` | Destructive — `-Confirm:$false` if prompted. |
| `ln -s target link` | `New-Item -ItemType SymbolicLink -Path link -Target target` | Requires elevation on Windows (unless Developer Mode is on). |
| `chmod +x file` | `icacls file /grant Users:RX` / `Set-ItemProperty` | Windows ACLs, not POSIX bits — use `Get-Acl` / `Set-Acl`. |
| `stat file` | `Get-Item file \| Format-List *` | |
| `du -sh dir` | `(Get-ChildItem dir -Recurse \| Measure-Object Length -Sum).Sum / 1MB` | Build your own formatting. |
| `df -h` | `Get-PSDrive -PSProvider FileSystem` / `Get-CimInstance Win32_LogicalDisk` | |

## 3. Text processing

| Bash | PowerShell | Notes |
| --- | --- | --- |
| `grep pattern file` | `Select-String pattern file` | Alias: `sls`. Returns `MatchInfo` objects (`LineNumber`, `Line`, `Matches`). |
| `grep -r pattern .` | `Get-ChildItem -Recurse \| Select-String pattern` | Or `Select-String pattern -Path **\*` in PS 7. |
| `grep -i pattern` | `Select-String -Pattern pattern` | Case-insensitive by default (unlike `grep`). Use `-CaseSensitive` to match `grep -i`-opposite. |
| `grep -v pattern` | `Select-String pattern -NotMatch` | |
| `grep -c pattern file` | `(Select-String pattern file).Count` | Or `... \| Measure-Object`. |
| `grep -l pattern *` | `Select-String pattern * \| Select-Object -Unique Path` | |
| `grep -C 2 pattern file` | `Select-String pattern file -Context 2,2` | |
| Filter on **object property** (no grep needed) | `Where-Object { $_.Status -eq 'Running' }` | Alias: `?`. This is the PS way — filter before formatting. |
| `sed 's/foo/bar/g' file` | `(Get-Content file) -replace 'foo','bar' \| Set-Content file` | `-replace` is regex. Use `[regex]::Escape()` for literal. |
| `sed -i 's/foo/bar/' file` | Same as above with `Set-Content` back to the file | |
| `awk '{print $2}'` | `\| ForEach-Object { ($_ -split '\s+')[1] }` | Or use `-Property` on objects instead of splitting strings. |
| `awk -F: '{print $1}' /etc/passwd` | `Get-Content /etc/passwd \| ForEach-Object { ($_ -split ':')[0] }` | |
| `cut -f2 -d,` | `Import-Csv file -Delimiter ',' \| Select-Object -ExpandProperty <col>` | CSV-aware > string-split. |
| `head -n 10` | `\| Select-Object -First 10` | |
| `tail -n 10` | `\| Select-Object -Last 10` | |
| `sort` | `Sort-Object` | Sort by property: `Sort-Object -Property Length`. |
| `sort -u` / `uniq` | `Sort-Object -Unique` / `Get-Unique` | `Get-Unique` requires sorted input. |
| `wc -l` | `\| Measure-Object -Line` | `.Lines`, `.Words`, `.Characters`. |
| `tr 'a-z' 'A-Z'` | `.ToUpper()` / `-creplace` | On strings — PS handles casing via methods. |
| `tee file` | `\| Tee-Object file` | |
| `xxd` / `hexdump` | `Format-Hex file` | |

## 4. Pipes and redirection

| Bash | PowerShell | Notes |
| --- | --- | --- |
| `cmd1 \| cmd2` | `cmd1 \| cmd2` | Identical syntax. **Semantics differ:** PS pipes objects. |
| `cmd > file` | `cmd > file` / `cmd \| Out-File file` | `Out-File` defaults to UTF-16 LE (5.1) / UTF-8 (7+); use `-Encoding utf8` explicitly. |
| `cmd >> file` | `cmd >> file` | |
| `cmd 2> err` | `cmd 2> err` | |
| `cmd > /dev/null` | `cmd > $null` / `cmd \| Out-Null` | **Never `> nul`** — creates a literal file on Windows. |
| `cmd 2>&1` | `cmd 2>&1` | On PS 5.1 invoking **native exe**, avoid this — wraps stderr lines as `NativeCommandError`s. |
| `cmd < input` | `Get-Content input \| cmd` | No `<` redirection for cmdlets. |
| `cmd1 && cmd2` | `cmd1; if ($?) { cmd2 }` | **PS 5.1 only.** PS 7+ supports `&&` and `\|\|` natively. |
| `cmd1 \|\| cmd2` | `cmd1; if (-not $?) { cmd2 }` | PS 7+: native `\|\|`. |
| `$(cmd)` | `$(cmd)` | Same syntax. |
| `` `cmd` `` | `$(cmd)` | PS has no backtick subshell (backtick is the escape char). |
| `<(cmd)` | temp file pattern | PS has no process substitution; write to a temp file and pass the path. |
| `cmd &` | `Start-Job { cmd }` / `Start-ThreadJob { cmd }` | Job objects, fetch output with `Receive-Job`. |
| `nohup cmd &` | `Start-Process pwsh -ArgumentList "-c","cmd" -WindowStyle Hidden` | For fully detached. |

## 5. Variables and expansions

| Bash | PowerShell | Notes |
| --- | --- | --- |
| `VAR=value` | `$var = 'value'` | No spaces around `=` in Bash; PS is flexible. |
| `export VAR=value` | `$env:VAR = 'value'` | Session-scope only. For persistent: `[Environment]::SetEnvironmentVariable('VAR','value','User')`. |
| `echo $VAR` | `$var` / `Write-Output $var` | |
| `echo "$VAR"` | `"$var"` | Double-quotes interpolate in both. |
| `echo '$VAR'` | `'$var'` | Single-quotes literal in both. |
| `${VAR:-default}` | `if ($var) { $var } else { 'default' }` | PS 7+: `$var ?? 'default'`. |
| `${VAR:=default}` | `$var = $var ?? 'default'` (PS 7+) | |
| `${#VAR}` | `$var.Length` | |
| `${VAR:6:5}` | `$var.Substring(6,5)` | |
| `${VAR^^}` / `${VAR,,}` | `$var.ToUpper()` / `$var.ToLower()` | |
| `${VAR/foo/bar}` | `$var -replace 'foo','bar'` | |
| `$?` | `$?` (bool) / `$LASTEXITCODE` (int, for native exes) | `$?` tracks last **statement** success; `$LASTEXITCODE` tracks last **native exe** exit code. |
| `$$` | `$PID` | |
| `$0` / `$1` / `$@` | `$PSCommandPath` / `$args[0]` / `$args` | Or use `param()` block for named params. |
| `set -e` | `$ErrorActionPreference = 'Stop'` | |
| `set -u` | `Set-StrictMode -Version Latest` | |
| `set -x` | `Set-PSDebug -Trace 1` | |

## 6. Control flow

| Bash | PowerShell |
| --- | --- |
| `if [ $n -eq 1 ]; then …; fi` | `if ($n -eq 1) { … }` |
| `if [[ a && b ]]` | `if ($a -and $b)` |
| `if [[ a \|\| b ]]` | `if ($a -or $b)` |
| `elif` | `elseif` |
| `for f in *.txt; do …; done` | `foreach ($f in Get-ChildItem *.txt) { … }` |
| `for i in {1..10}` | `1..10 \| ForEach-Object { … }` or `for ($i=1; $i -le 10; $i++) { … }` |
| `while read line; do …; done < file` | `Get-Content file \| ForEach-Object { $line = $_; … }` |
| `case $x in a) …;; esac` | `switch ($x) { 'a' { … } default { … } }` |
| `break` / `continue` | Same keywords. |

Comparison operators are **word-style** in PS: `-eq`, `-ne`, `-lt`, `-gt`, `-le`, `-ge`, `-match` (regex), `-like` (wildcard), `-in`, `-contains`. `==` / `!=` don't work.

## 7. Functions and params

```bash
greet() {
  local name=$1
  echo "Hi, $name"
}
greet "Ada"
```

```powershell
function Greet {
    [CmdletBinding()]
    param([Parameter(Mandatory)][string]$Name)
    "Hi, $Name"
}
Greet -Name 'Ada'
# positional also works: Greet 'Ada'
```

Key differences:
- PS functions emit to the **pipeline** — any expression that's not captured becomes output. Use `[void]` or `| Out-Null` to discard.
- Always prefer `param()` + validation attributes over `$args[n]` — makes tab-completion, help, and pipeline input work.
- `return` in PS **exits** the function (and emits the value). In Bash, `return N` is the exit status (0-255).

## 8. Arrays and hashes

| Bash | PowerShell |
| --- | --- |
| `arr=(a b c)` | `$arr = @('a','b','c')` |
| `${arr[0]}` | `$arr[0]` |
| `${arr[@]}` | `$arr` |
| `${#arr[@]}` | `$arr.Count` / `$arr.Length` |
| `arr+=(d)` | `$arr += 'd'` (creates new array; use `[System.Collections.Generic.List[string]]` for append-heavy work) |
| `declare -A map`<br>`map[key]=val` | `$map = @{}; $map['key'] = 'val'` |
| `${map[key]}` | `$map['key']` or `$map.key` |
| `${!map[@]}` | `$map.Keys` |

## 9. Processes and system

| Bash | PowerShell |
| --- | --- |
| `ps aux` | `Get-Process` (alias `ps`) |
| `ps -ef \| grep foo` | `Get-Process foo` / `Get-Process \| Where-Object Name -like '*foo*'` |
| `kill PID` | `Stop-Process -Id PID` (alias `kill`) |
| `kill -9 PID` | `Stop-Process -Id PID -Force` |
| `pkill name` | `Stop-Process -Name name` |
| `top` / `htop` | `Get-Process \| Sort-Object CPU -Descending \| Select-Object -First 20` |
| `uptime` | `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` → compute delta |
| `hostname` | `$env:COMPUTERNAME` / `hostname` |
| `whoami` | `whoami` / `$env:USERNAME` |
| `uname -a` | `Get-CimInstance Win32_OperatingSystem \| Format-List *` |
| `which cmd` | `Get-Command cmd` (alias `gcm`) |
| `env` | `Get-ChildItem env:` |
| `history` | `Get-History` (in-session) + `(Get-PSReadLineOption).HistorySavePath` (file) |

## 10. Networking

| Bash | PowerShell |
| --- | --- |
| `curl URL` | `Invoke-WebRequest URL` (alias `iwr`) |
| `curl -o file URL` | `Invoke-WebRequest URL -OutFile file` |
| `curl -X POST -d json URL` | `Invoke-RestMethod -Method Post -Body $json -ContentType 'application/json' -Uri URL` |
| `wget URL` | `Invoke-WebRequest URL -OutFile (Split-Path URL -Leaf)` |
| `ping host` | `Test-Connection host -Count 4` |
| `nslookup host` | `Resolve-DnsName host` |
| `netstat -an` | `Get-NetTCPConnection` / `Get-NetUDPEndpoint` |
| `ssh user@host cmd` | `Invoke-Command -HostName host -UserName user -ScriptBlock { cmd }` (PS 7 SSH) / `Enter-PSSession` |
| `scp src user@host:dst` | `Copy-Item src -ToSession (New-PSSession -HostName host -UserName user) -Destination dst` |

## 11. JSON / CSV / XML

| Bash | PowerShell |
| --- | --- |
| `jq '.key'` | `$obj = $json \| ConvertFrom-Json; $obj.key` |
| `jq '.items[].name'` | `$obj.items.name` (auto-flattens) |
| `echo $data \| jq -n` | `$data \| ConvertTo-Json -Depth 10` |
| `awk -F, '{print $1}' f.csv` | `Import-Csv f.csv \| Select-Object -ExpandProperty <Col>` |
| XSLT / `xmllint` | `$xml = [xml](Get-Content f.xml); $xml.SelectNodes('//x')` |

## 12. Quoting rules — the #1 gotcha

| Thing | Bash | PowerShell |
| --- | --- | --- |
| Single quotes | **Literal** — no expansion | **Literal** — no expansion |
| Double quotes | Expands `$var`, `` `cmd` ``, `\` | Expands `$var`, `$(expression)`, backticks are **escape char** |
| Escape inside double quotes | `\` | `` ` `` (backtick) |
| Heredoc | `<<EOF … EOF` | `@"`…`"@` (interp) / `@'`…`'@` (literal). Closing marker MUST be at column 0. |
| `\n` / `\t` | In `$'...'` or with `echo -e` | `` `n `` / `` `t `` inside `"…"` only |

## 13. Windows-specific gotchas when mixing Bash and PowerShell

- **`/dev/null` ≠ `nul`**. In Git Bash, `> nul` creates a literal file. Always `/dev/null` in Bash, `$null` / `Out-Null` in PS.
- **Backslash paths** in Bash: always quote them (`"C:\Users\foo"`) or use forward slashes (`C:/Users/foo`). Unquoted, Bash eats the backslash.
- **`powershell -Command` via Bash** loses the object pipeline and UTF-16 LE garbles output. Use the PowerShell tool directly (blocked by the `pwsh-native` hook).
- **`cd && powershell …`** — shell context is lost. Keep the whole operation in one tool call.
- **`wmic` is dead** — removed in Windows 11. Always `Get-CimInstance`.
- **`Get-WmiObject` is dead** — removed in PS 7. Always `Get-CimInstance`.
- **`Get-EventLog` is dead** — replaced by `Get-WinEvent` (supports modern logs).
- **Encoding:** PS 5.1 defaults to UTF-16 LE with BOM for file output. Always pass `-Encoding utf8` unless you *want* UTF-16.
- **Aliases may vary:** `curl` and `wget` in PS 5.1 aliased to `Invoke-WebRequest` (different output shape than real `curl`). In PS 7, aliases are **removed** and real `curl.exe` / `wget.exe` (if installed) take over. If writing cross-version scripts, use `Invoke-WebRequest` / `Invoke-RestMethod` explicitly.

## Sources

- [TheShellNut — PowerShell equivalents for common Linux/bash commands](https://mathieubuisson.github.io/powershell-linux-bash/)
- [Ironman Software — Bash vs PowerShell Cheat Sheet](https://blog.ironmansoftware.com/daily-powershell/bash-powershell-cheatsheet)
- [Secure Ideas — From Linux to PowerShell and Back](https://www.secureideas.com/blog/from-linux-to-powershell-and-back-a-quick-command-reference)
- [TechTarget — Try out PowerShell grep equivalents](https://www.techtarget.com/searchitoperations/tutorial/Try-out-PowerShell-grep-equivalents)
- [Scott Hanselman — Sed, Grep, Awk, Cut in PowerShell](https://www.hanselman.com/blog/unix-fight-sed-grep-awk-cut-and-pulling-groups-out-of-a-powershell-regular-expression-capture)

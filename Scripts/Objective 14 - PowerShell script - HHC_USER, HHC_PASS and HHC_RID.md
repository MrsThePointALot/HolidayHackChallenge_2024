
```
function set_environment_variable($a, $b) {[System.Environment]::SetEnvironmentVariable($a,$b,'User')}
set_environment_variable 'HHC_USER' 'HonestDon@whitehouse.gov'
set_environment_variable 'HHC_PASS' 'ComeAtMeBro2024!'

[System.Environment]::GetEnvironmentVariable('HHC_USER','User')
[System.Environment]::GetEnvironmentVariable('HHC_PASS','User')

```

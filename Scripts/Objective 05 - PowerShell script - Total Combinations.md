```
$digits = 2,6,7,8

$combinations = foreach ($d1 in $digits) {
    foreach ($d2 in $digits) {
        foreach ($d3 in $digits) {
            foreach ($d4 in $digits) {
                foreach ($d5 in $digits) {
                    $combo = "$d1$d2$d3$d4$d5"
                    # Check if all required digits are present
                    if (($combo.Contains("2")) -and 
                        ($combo.Contains("6")) -and 
                        ($combo.Contains("7")) -and 
                        ($combo.Contains("8"))) {
                        $combo
                    }
                }
            }
        }
    }
}
Write-Host "Total combinations:" $combinations.Count

```

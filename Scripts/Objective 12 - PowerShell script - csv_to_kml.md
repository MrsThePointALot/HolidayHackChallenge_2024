```
function csv_to_kml ($csv_filename) {
	# Read the CSV file
	$csvData = Import-Csv -Path $csv_filename

	# Create KML header
	$kml = @"
	<?xml version="1.0" encoding="UTF-8"?>
	<kml xmlns="http://www.opengis.net/kml/2.2">
	  <Document>
		<name>Drone Flight Path</name>
		<Style id="yellowLine">
		  <LineStyle>
			<color>7f00ffff</color>
			<width>4</width>
		  </LineStyle>
		</Style>
		<Placemark>
		  <name>Flight Path</name>
		  <styleUrl>#yellowLine</styleUrl>
		  <LineString>
			<extrude>1</extrude>
			<tessellate>1</tessellate>
			<altitudeMode>absolute</altitudeMode>
			<coordinates>
	"@

	# Add coordinates from CSV
	foreach ($line in $csvData) {
		$kml += "          $($line.PSObject.Properties["OSD.longitude"].Value),$($line.PSObject.Properties["OSD.latitude"].Value),$($line.PSObject.Properties["OSD.altitude [ft]"].Value)`n"
	}

	# Add KML footer
	$kml += @"
			</coordinates>
		  </LineString>
		</Placemark>
	  </Document>
	</kml>
	"@

	# Save to file
	$dstfile = "$((Get-Item $csv_filename).Basename).kml"
	$kml | Out-File $dstfile -Encoding UTF8
	Write-Host "KML file created successfully: $dstfile"
}

csv_to_kml "ELF-HAWK-dump.csv"

```
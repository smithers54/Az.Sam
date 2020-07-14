# Azure Route Table Route Addition Script

## Description

A powershell script for adding routes from a text file. Replace the $path with your txt file that has your prefixes. The prefixes must be in a line break list.

Ex.

10.1.0.0/16
10.2.1.0/24
10.3.3.4/32

The script will prompt for a "tag" that get's appended to the route name, the next hop type, resource group and route table name.

``` powershell

#------------------------------------------------------------------------------   
#   
#    
# THIS CODE AND ANY ASSOCIATED INFORMATION ARE PROVIDED â€œAS ISâ€ WITHOUT   
# WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT   
# LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS   
# FOR A PARTICULAR PURPOSE. THE ENTIRE RISK OF USE, INABILITY TO USE, OR    
# RESULTS FROM THE USE OF THIS CODE REMAINS WITH THE USER.   
#   
#------------------------------------------------------------------------------  

# Writes the prefixes to a txt file in the C:\temp drive. The text file must be a line break list of prefixes
# Example:
# 1.1.1.1/32
# 10.3.5.0/24

$path = "C:\temp\addressspaces.txt" ## Add prefix txt file path here
$tag = Read-Host "Enter a tag to be appened to the routes name (Ex. OnPrem2NVA)"

# Prompts the user for Next Hop type (Credit to Ishan Shukla)
$nexthoptype = Read-Host "Enter the corresponding number for the next hop: `n 1.Internet `n 2.None `n 3.VirtualAppliance `n 4.VirtualNetworkGateway `n 5.VnetLocal" $str "`n"
if($nexthoptype -eq 1){$nexthoptype = 'Internet'}
if($nexthoptype -eq 2){$nexthoptype = 'None'}
if($nexthoptype -eq 3){$nexthoptype = 'VirtualAppliance'}
if($nexthoptype -eq 4){$nexthoptype = 'VirtualNetworkGateway'}
if($nexthoptype -eq 5){$nexthoptype = 'VnetLocal'}
#$nexthoptype
if($nexthoptype -eq 'VirtualAppliance'){$nexthopIP = Read-Host "Enter IP address of the NVA"}

# Creates a CSV file that a foreach script can create separate routes from
$routes = Get-Content $path
$excel = New-Object -ComObject excel.application
$excel.visible = $False
$workbook = $excel.Workbooks.Add()
$diskSpacewksht= $workbook.Worksheets.Item(1)
$diskSpacewksht.Cells.Item(1,1) = 'Name'
$diskSpacewksht.Cells.Item(1,2) = 'AddressPrefix'

$col1 = 2
$col2 = 2


# The foreach loop that adds the txt values to the workbook.
 foreach ($routesVal in $routes){
              $diskSpacewksht.Cells.Item($col1,1) = ("$tag-")+($routesVal -replace '[/]','.')
              $col1++
}

 foreach ($routesVal in $routes){
              $diskSpacewksht.Cells.Item($col2,2) = $routesVal 
              $col2++
}

$excel.DisplayAlerts = 'False'
$ext=".xlsx"
$path="C:\temp\Routes2Add$tag$ext"
$workbook.SaveAs($path) 
$workbook.Close
$excel.DisplayAlerts = 'False'
$excel.Quit()


# Converts the workbook to a csv

$pathnew = "C:\temp\Routes2Add$tag"

$excelFile = $path
$Excel = New-Object -ComObject Excel.Application
$Excel.Visible = $false
$Excel.DisplayAlerts = $false
$wb = $Excel.Workbooks.Open($excelFile)
    foreach ($ws in $wb.Worksheets)
    {
        $ws.SaveAs("$pathnew" + $File + ".csv", 6)
    }
    $Excel.Quit()


# Runs a foreach loop to add all of the prefixes in the csv. The name of the route will be it's Service Tag + prefix
# You will be prompted for Resource Group and the Route table name
$rg = Read-Host -Prompt 'Resource Group Name'
$rtName = Read-Host -Prompt 'Route Table Name'


$routeTable = Get-AzRouteTable -ResourceName $rtName -ResourceGroupName $rg


foreach ($file in import-csv "c:\temp\Routes2Add$tag.csv")
{
    if ($nexthoptype -eq 'VirtualAppliance')
    {
    $routeTable | Add-AzRouteConfig -Verbose  `
                  -Name $file.Name `
                  -AddressPrefix $file.AddressPrefix `
                  -NextHopType $nexthoptype `
                  -NextHopIpAddress $nexthopIP
    }
    Else
    {
    $routeTable | Add-AzRouteConfig -Verbose  `
                  -Name $file.Name `
                  -AddressPrefix $file.AddressPrefix `
                  -NextHopType $nexthoptype
    }
}

$routeTable | Set-AzRouteTable

#Stops any background sessions of EXCEL
Stop-Process -Name EXCEL

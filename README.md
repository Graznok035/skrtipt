$csv = Import-Csv -Path "C:\kasutajad.csv"

foreach ($kasutaja in $csv) {
    # Eesnimi + perenimi lahutamine
    $nimeOsad = $kasutaja.Name.Trim().Split(" ")
    if ($nimeOsad.Count -lt 2) {
        Write-Host "Nimi '$($kasutaja.Name)' pole kujul 'Eesnimi Perenimi', jätan vahele." -ForegroundColor Red
        continue
    }
    $eesnimi = $nimeOsad[0]
    $perenimi = $nimeOsad[1]
    $kasutajanimi = ($eesnimi[0].ToString() + $perenimi).ToLower()

    $ouPath = "OU=$($kasutaja.OU),OU=KASUTAJAD,DC=kohvi,DC=praktika"

    # Kontrolli kas OU on olemas, kui ei, loo see
    try {
        Get-ADOrganizationalUnit -Identity $ouPath -ErrorAction Stop
    } catch {
        Write-Host "OU '$($kasutaja.OU)' puudub, loon..." -ForegroundColor Yellow
        New-ADOrganizationalUnit -Name $kasutaja.OU -Path "OU=KASUTAJAD,DC=kohvi,DC=praktika"
    }

    # Kontrolli kas grupp on olemas, kui ei, loo see
    $grupp = $kasutaja.OU
    try {
        Get-ADGroup -Identity $grupp -ErrorAction Stop
    } catch {
        Write-Host "Grupp '$grupp' puudub, loon..." -ForegroundColor Yellow
        New-ADGroup -Name $grupp -SamAccountName $grupp -GroupScope Global -Path $ouPath
    }

    # Loo kasutaja kui pole juba olemas
    if (-not (Get-ADUser -Filter {SamAccountName -eq $kasutajanimi} -ErrorAction SilentlyContinue)) {
        New-ADUser `
            -Name $kasutaja.Name `
            -GivenName $eesnimi `
            -Surname $perenimi `
            -SamAccountName $kasutajanimi `
            -UserPrincipalName "$kasutajanimi@kohvi.praktika" `
            -AccountPassword (ConvertTo-SecureString $kasutaja.Password -AsPlainText -Force) `
            -Path $ouPath `
            -Enabled $true
        Write-Host "Kasutaja '$kasutajanimi' loodud." -ForegroundColor Green
    } else {
        Write-Host "Kasutaja '$kasutajanimi' on juba olemas, jätan vahele." -ForegroundColor Cyan
    }

    # Lisa kasutaja gruppi
    $adKasutaja = Get-ADUser -Identity $kasutajanimi -ErrorAction SilentlyContinue
    if ($adKasutaja) {
        Add-ADGroupMember -Identity $grupp -Members $adKasutaja
    }
}

# Еще один способ увидеть коммуникации приложений

Добрый день, коллеги. Как известно, есть очень полезная утилита - [sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon). В двух словах, она позволяет вам собирать и "логировать" события, происходяшие в Windows. Одним из таких событий является попытка установить сетевое соединение. Таким образом, можно попытаться узнать, куда ходят ваши приложения. Для этого нам понадобятся:

- сам [sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- конфигурация к нему, я предпочитаю использовать вот [эту](https://github.com/SwiftOnSecurity/sysmon-config)
- модуль [PSQuickGraph module](https://www.powershellgallery.com/packages/PSQuickGraph/1.1)
- и немного фантазии

<cut />
В принципе, фантазии нам надо совсем чуть-чуть. Sysmon пишет события в лог ```Microsoft-Windows-Sysmon/Operational```. Значит нам надо их оттуда достать, разобрать и отобразить. Примерно вот так:

```powershell
$ids = Get-WinEvent -LogName Microsoft-Windows-Sysmon/Operational | ? {$_.id -eq 3}
$commObjects = $ids | % {
    New-Object psobject -Property @{ 
        RuleName            = $_.Properties[0].value
        UtcTime             = $_.Properties[1].value
        ProcessGuid         = $_.Properties[2].value
        ProcessId           = $_.Properties[3].value
        Image               = $_.Properties[4].value
        User                = $_.Properties[5].value
        Protocol            = $_.Properties[6].value
        Initiated           = $_.Properties[7].value
        SourceIsIpv6        = $_.Properties[8].value
        SourceIp            = $_.Properties[9].value
        SourceHostname      = $_.Properties[10].value
        SourcePort          = $_.Properties[11].value
        SourcePortName      = $_.Properties[12].value
        DestinationIsIpv6   = $_.Properties[13].value
        DestinationIp       = $_.Properties[14].value
        DestinationHostname = $_.Properties[15].value
        DestinationPort     = $_.Properties[16].value
        DestinationPortName = $_.Properties[17].value
        SourceString   = "$($_.Properties[4].value)`:$($_.Properties[3].value)"
        DestinationString   = "$($_.Properties[14].value)`:$($_.Properties[16].value)"
    }
}

$g = New-Graph -Type BidirectionalGraph
$commObjects | % {
    Add-Edge -From $_.SourceString -To $_.DestinationString -Graph $g | Out-Null
}

Show-GraphLayout -Graph $g
```

К несчастью, значения в свойстве ```Properties``` в виде списка, просто значения, без ключей. Потому, чтобы их связать, пришлось поступить грубо. В конечном счете, мы просто берем эти значения из каждой записи "лога", преобразуем их в объекты, а затем добавляем в граф как вершины и отображаем.

Важно помнить, что процесс с одним и тем же "путем", может быть запущен много раз. С другой стороны, вершина с одним и тем же именем не добавляется дважды. Потому, чтобы уникально представить каждый процесс на графе, мы немного модифицируем оригинальный набор значений, добавляя два новых. Это дает нам возможность точно идентифицировать процесс, поскольку его идентификатор - значение относительно уникальное.

```powershell
        SourceString   = "$($_.Properties[4].value)`:$($_.Properties[3].value)"
        DestinationString   = "$($_.Properties[14].value)`:$($_.Properties[16].value)"
```

Вот так это может выглядеть в конечном итоге

![sysmonlognetgraph](/images/2018-10-04-19-54-43.png)

Надеюсь, это кому-нибудь пригодится
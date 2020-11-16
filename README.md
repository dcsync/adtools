Tools for parsing and filtering
[PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/dev) output.

ParseDomain
===========

ParseDomain parse and filters AD dumps. It can digest output from DumpDomain,
ldapdomaindump, and PowerView's with `Get-Domain*` functions ( e.g.
`Get-DomainComputer`, `Get-DomainObject`). Some examples:

Searching for Windows 7 machines:

    $ ./ParseDomain --regex 'operatingsystem=.*windows 7.*' --property name --property operatingsystem --no-headers output.txt
    Item:
      - name: CORP339
      - operatingsystem: Windows 7 Enterprise

    Item:
      - name: TEST2
      - operatingsystem: Windows 7 Enterprise

    Item:
      - name: ITML5
      - operatingsystem: Windows 7 Enterprise

Searching for domain controllers:

    $ ./ParseDomain --regex 'distinguishedname=.*OU=Domain Controllers.*' --property name --property operatingsystem output.txt
    Item:
      - name: DC2
      - operatingsystem: Windows Server 2012 R2 Datacenter

    Item:
      - name: DC1
      - operatingsystem: Windows Server 2012 R2 Datacenter

ParseDomain can make a user-group tree:

    $ ./ParseDomain --tree output.txt
    ...
    Denied RODC Password Replication Group
        Read-only Domain Controllers
        Domain Controllers
        Domain Admins
            ...
            Administrator
        Schema Admins
            ...
            Administrator
        Cert Publishers
            CORPDC
    ...

We recommend the following PowerShell example to properly dump a domain with PowerView.
Replace `Get-DomainObject` with more selective functions if desired.

	$FormatEnumerationLimit=-1
	Get-DomainObject | Format-List -Property * > objects.domain

For more options:

    $ ./ParseDomain -h
    usage: ParseDomain [-h] [-p PROPERTY] [-s SORT] [-c OBJECT_CLASS]
                       [--parent-class PARENT_CLASS] [-f FILTER] [-F FILTER_FILE]
                       [-r REGEX] [-H] [--emails] [--tree] [--admins]
                       files [files ...]
    
    positional arguments:
      files                 Files to parse
    
    optional arguments:
      -h, --help            show this help message and exit
      -p PROPERTY, --property PROPERTY
                            Select property
      -s SORT, --sort SORT  Sort by field
      -c OBJECT_CLASS, --object-class OBJECT_CLASS
                            Select objects of class. May be case-insensitive AD
                            class names or aliases gpo=groupPolicyContainer,
                            ou=organizationalUnit, scp=serviceConnectionPoint,
                            td=trustedDomain, u=user, g=group, c=computer
      --parent-class PARENT_CLASS
                            Select objects in class, including parent classes. May
                            be case-insensitive AD class names or aliases (see
                            --object-class)
      -f FILTER, --filter FILTER
                            Filter by property (property=value)
      -F FILTER_FILE, --filter-file FILTER_FILE
                            Filter by properties in file (property=value)
      -r REGEX, --regex REGEX
                            Filter by property (property=regex)
      -H, --no-headers      Don't show headers
      --emails              Produce a list of emails (for Get-DomainUser)
      --tree                Produce a tree (for Get-DomainGroup)
      --admins              Only show admins (admincount=1)

ParseUserHunter
===============

ParseUserHunter can parse and filter the output of
`Invoke-UserHunter`/`Find-DomainUserLocation`. Some examples:

Grouping by user (default):

    $ ./ParseUserHunter output.txt
    --- DA_bob ---
    2 sessions of CORP\DA_bob@SQL2.corp.host (IP 10.10.5.6)
    
    --- dave ---
    2 sessions of CORP\dave@FILES.corp.host (IP 10.10.9.3)
    
    ...

Grouping by computer:

    $ ./ParseUserHunter --by-computer output.txt
    --- ENT6.corp.host ---
    2 sessions of CORP\mark@ENT6.corp.host (IP 10.10.5.6)
    
    --- FILES.corp.host ---
    2 sessions of CORP\mark@SQL2.corp.host (IP 10.10.5.6)
    2 sessions of CORP\dave@FILES.corp.host (IP 10.10.9.3)
    3 sessions of CORP\chris@SQL2.corp.host (IP 10.10.5.6)

    ...

Filtering by a computer:

    $ ./ParseUserHunter -c IT-SERVER output.txt
    --- erin ---
    2 sessions of CORP\eric@IT-SERVER.corp.host (IP 10.10.3.3)

    --- IT_Admin ---
    2 sessions of CORP\IT_Admin@IT-SERVER.corp.host (IP 10.10.3.3)

For more options:

    $ ./ParseUserHunter
    usage: ParseUserhunter [-h] [-u USER] [--user-file USER_FILE] [-c COMPUTER]
                           [--computer-file COMPUTER_FILE] [-C]
                           file
    
    positional arguments:
      file                  File to parse
    
    optional arguments:
      -h, --help            show this help message and exit
      -u USER, --user USER  Filter by user
      --user-file USER_FILE
                            Filter by users from file
      -c COMPUTER, --computer COMPUTER
                            Filter by computer
      --computer-file COMPUTER_FILE
                            Filter by computers from file
      -C, --by-computer     Sort by computer instead of by user

ParseKerberoast
===============

ParseKerberoast extracts the hash parts from the output of `Invoke-Kerberoast`
so you can feed them to Hashcat.

    $ head -5 kerberoast.txt
    SamAccountName       : bobby
    DistinguishedName    : CN=Bobby Smith,CN=Users,DC=corp,DC=host
    ServicePrincipalName : MSSQLSvc/sql.corp.host:12345
    TicketByteHexStream  : 
    Hash                 : $krb5tgs$23$*bobby$corp.host$MSSQLSvc/sql.corp.host:12345*$A129812C1DF12

    $ ./ParseKerberoast kerberoast.txt
    $krb5tgs$23$*bobby$corp.host$MSSQLSvc/sql.corp.host:12345*$A129812C1DF12AF55325BB32598C199BBA10
    ....

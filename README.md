DumpLDAP
========

DumpLDAP dumps an LDAP server to json. This allows for offline exploration and better network opsec. Usage:

    $ ./DumpLDAP -u CORP/DA_david -p P@ssw0rd 10.51.5.3 -o dump.json
    [.] connecting to server
        - host: 10.51.5.3
    [.] authenticating
        - user: CORP\DA_david
        - password: P@ssw0rd
        - auth type: NTLM
	...

Full options:

    usage: DumpLDAP [-h] -u USER -p PASSWORD [--simple-auth] [-q QUERY] [-b BASE]
                    [--page-size PAGE_SIZE] [--page-throttle PAGE_THROTTLE]
                    [--no-pagination] [-o OUTPUT]
                    host
    
    auth:
      -u USER, --user USER  Auth username (DOMAIN\user)
      -p PASSWORD, --password PASSWORD
                            Auth password or LM:NTLM hash
      --simple-auth         Use SIMPLE auth instead of NTLM
    
    ldap:
      host                  LDAP server
      -q QUERY, --query QUERY
                            LDAP query (default: dump everything with
                            (&(objectclass=*)))
      -b BASE, --base BASE  LDAP base (default: AD root)
      --page-size PAGE_SIZE
                            LDAP request page size (default: 100)
      --page-throttle PAGE_THROTTLE
                            Delay between LDAP page requests (default: 0.0)
      --no-pagination       Do not use LDAP pagination
    
    output:
      -o OUTPUT, --output OUTPUT
                            Output file (default: stdout)

ParseDomain
===========

ParseDomain parse and filters AD dumps. It can digest output from DumpDomain,
ldapdomaindump, and PowerView's with `Get-Domain*` functions ( e.g.
`Get-DomainComputer`, `Get-DomainObject`). Some examples:

Searching for Windows 7 machines:

    $ ./ParseDomain --regex 'operatingsystem=.*windows 7.*' --property name --property operatingsystem --no-headers dump.json
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

    $ ./ParseDomain --regex 'distinguishedname=.*OU=Domain Controllers.*' --property name --property operatingsystem dump.json
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

    usage: ParseDomain [-h] [--no-generator] [-f FILTER] [-F FILTER_FILE]
                       [-r REGEX] [-P PYTHON_FILTER] [-n] [-a] [-c CATEGORY]
                       [-q {laps,admin,dc,da,server}] [-p PROPERTY] [-s SORT]
                       [--pretty-print] [--no-keys] [--show-binary] [-D]
                       [--emails] [--tree] [--count] [--wordlist]
                       files [files ...]
    
    optional arguments:
      -h, --help            show this help message and exit
    
    parsing:
      --no-generator        Do not use generator-producing lazy-loading JSON
                            performance hack that works most of the time but not
                            always
    
    filters:
      -f FILTER, --filter FILTER
                            Filter by property (property=value)
      -F FILTER_FILE, --filter-file FILTER_FILE
                            Filter by properties in file (property=value)
      -r REGEX, --regex REGEX
                            Filter by property (property=regex)
      -P PYTHON_FILTER, --python-filter PYTHON_FILTER
                            Filter with an evaled Python boolean expression. Each
                            run exposes "item" which is {key: value, ...}
      -n, --not             Invert the filters
      -a, --and             Treat filters as an "and" instead of an "or". Narrow
                            search down for search filter applied.
      -c CATEGORY, --category CATEGORY
                            Filter objects by x-baseCategory. Aliases: {'gpo':
                            'Group-Policy-Container', 'ou': 'Organizational-Unit',
                            'scp': 'Service-Connection-Point', 'td': 'Trusted-
                            Domain', 'p': 'Person', 'g': 'Group', 'c': 'Computer'}
      -q {laps,admin,dc,da,server}, --quick-filter {laps,admin,dc,da,server}
                            Add a quick filter. Options: laps, admin, dc, da,
                            server
    
    regular output:
      -p PROPERTY, --property PROPERTY
                            Select property to show
      -s SORT, --sort SORT  Sort by field
      --pretty-print        Show pretty human-readable headers
      --no-keys             Do not show parameter keys
      --show-binary         Show binary blobs in their encoded formats
      -D, --debug           Enable debug output
    
    special output:
      --emails              Produce a list of user emails
      --tree                Produce a user-group tree (for Get-DomainGroup)
      --count               Produce tally of items
      --wordlist            Produce a wordlist from field values
      files                 Files to parse

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

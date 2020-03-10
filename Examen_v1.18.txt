#########################################################                           
### DACL		                                      ###
### Syntra Mechelen                                   ###
### Academiejaar 2019-2020                            ###
### 08/03/2020 - Versie 1.18                          ###
### Exampenopdracht DSC (Desired State Configuration) ###
#########################################################


Configuration BMOOS_Config
{
    param
	(
		[Parameter(Mandatory = $true)]
		[ValidateNotNullOrEmpty()]
		[System.Management.Automation.PSCredential]
		$Credential,

		[Parameter(Mandatory = $true)]
		[ValidateNotNullOrEmpty()]
		[System.Management.Automation.PSCredential]
		$SafeModePassword
	)
    
    ### POWERSHELL DSC MODULES TO BE IMPORTED 
    Import-DscResource -Module NetworkingDsc -ModuleVersion 7.4.0.0
    Import-DscResource -Module ComputerManagementDsc -ModuleVersion 7.1.0.0
    Import-DscResource -ModuleName ActiveDirectoryDsc -ModuleVersion 5.0.0
    Import-DscResource -ModuleName 'PSDesiredStateConfiguration'
    #BOVENSTAANDE REGEL TOEVOEGEN OM ONDERSTAANDE FOUTMELDING TE VOORKOMEN
    #   WARNING: The configuration 'BMOOS_Config' is loading one or more built-in resources without exp
    #   licitly importing associated modules. Add Import-DscResource –ModuleName 'PSDesiredStateConfigu
    #   ration' to your configuration to avoid this message.
	Import-DscResource -ModuleName xPSDesiredStateConfiguration -ModuleVersion 9.0.0
    Import-DscResource -Module Xdhcpserver -ModuleVersion 2.0.0.0
    Import-DscResource -Module xSmbShare


	Node $allnodes.nodename
	{

        ### STATIC IP ADDRESS
        ### Module : NetworkingDsc
	    ### https://github.com/dsccommunity/NetworkingDsc/tree/master/source/Examples/Resources/IPAddress
	
        IPAddress NewIPv4Address
        {
            IPAddress      = '192.168.1.9/24'
            InterfaceAlias = 'Ethernet'
            AddressFamily  = 'IPV4'
        }


        ### SET DEFAULT GATEWAY
        ### Module : NetworkingDsc
	    ### https://github.com/dsccommunity/NetworkingDsc/blob/master/source/Examples/Resources/DefaultGatewayAddress/2-DefaultGatewayAddress_SetDefaultGateway_Config.ps1
	
        DefaultGatewayAddress SetDefaultGateway
        {
            Address        = '192.168.1.1'
            InterfaceAlias = 'Ethernet'
            AddressFamily  = 'IPv4'
            DependsOn      = '[IPAddress]NewIPv4Address'
        }


        ### DNS Server Address
        ### Module : NetworkingDsc
        ### https://github.com/dsccommunity/NetworkingDsc/blob/master/source/Examples/Resources/DnsServerAddress/1-DnsServerAddress_Config.ps1
        ### https://github.com/dsccommunity/NetworkingDsc/wiki/DNSServerAddress

		
		### Onserdtaande zit nu onder XDhcpServerOption ServerOption
        #DnsServerAddress DnsServerAddress
        #{
        #    Address        = '192.168.1.9'
        #    InterfaceAlias = 'Ethernet'
        #    AddressFamily  = 'IPv4'
            #Validate      = $true   # Werkt wel zonder deze regel
        #    DependsOn      = '[DefaultGatewayAddress]SetDefaultGateway'
        #}


        ### RENAME COMPUTER IN DOMAIN CONFIGURATION
        ### Module : ComputerManagementDsc
        ### https://github.com/dsccommunity/ComputerManagementDsc

        Computer NewName
        {
            Name       = 'DC01'
            Credential = $Credential
            DependsOn  = '[IPAddress]NewIPv4Address'
        }


        ### ADDomain NewForest CONFIGURATION
		### Module : ActiveDirectoryDsc
		### https://www.powershellgallery.com/packages/ActiveDirectoryDsc/5.0.0
    
        WindowsFeature 'ADDS'
        {
            Name   = 'AD-Domain-Services'
            Ensure = 'Present'
        }
	
        WindowsFeature 'RSAT'
        {
            Name   = 'RSAT-AD-PowerShell'
            Ensure = 'Present'
        }

        ADDomain 'bmoos.local'
        {
            DomainName                    = 'bmoos.local'
            Credential                    = $Credential
            SafemodeAdministratorPassword = $SafeModePassword
            ForestMode                    = 'WinThreshold'
            DependsOn                     = '[Computer]NewName'
        }


        ### xDhcpServer CONFIGURATION
        ### Module : Xdhcpserver
		### https://github.com/dsccommunity/xDhcpServer

		WindowsFeature “dhcp-server”
		{
			Name      = “DHCP”
			Ensure    = “Present”
            DependsOn = '[ADDomain]bmoos.local'
		}

		WindowsFeature RSATDHCP
		{
			Name      = "RSAT-DHCP"
			Ensure    = "Present"
			DependsOn = '[windowsfeature]dhcp-server'
		}
				
		XDhcpServerScope “DHCPConfig”
		{
			ScopeId       = "192.168.1.0"
			IPendrange    = “192.168.1.100”
			IPStartrange  = “192.168.1.10”
			Subnetmask    = “255.255.255.0”
			Ensure        = “Present”
			Name          = “IPV4Scope”
			State         = “Active”
			LeaseDuration = ((New-TimeSpan -hours 8 ).ToString())
			AddressFamily = “IPV4”
	    	DependsOn     = "[Windowsfeature]dhcp-server"
        }

        XDhcpServerAuthorization Authorization
        {
            Ensure        = “Present”
            IPAddress     = "192.168.1.9" 
        }

        XDhcpServerOption ServerOption
        {
            ScopeID            = "192.168.1.0"
            Router             = "192.168.1.1"
            DnsServerIPAddress = "192.168.1.9"
            DnsDomain          = "bmoos.local"
            AddressFamily      = "IPv4"
            Ensure             = "Present"
        }


        ### SHARED DIRECTORIES
		### Module : xPSDesiredStateConfiguration
		### https://www.powershellgallery.com/packages/xPSDesiredStateConfiguration/9.0.0
        
        File Share 
        {							
            Ensure          = "Present"
            DestinationPath = "c:\Share"
            Type            = "Directory"
        }
        
        File Marketing 
        {
            Ensure          = "Present"
            DestinationPath = "c:\Share\Marketing"
            Type            = "Directory"
            
        }

        File HR 
        {
            Ensure          = "Present"
            DestinationPath = "c:\Share\HR"
            Type            = "Directory"
        }

        File Production 
        {
            Ensure          = "Present"
            DestinationPath = "c:\Share\Production"
            Type            = "Directory"
        }
 
        File Logistics 
        {
            Ensure          = "Present"
            DestinationPath = "c:\Share\Logistics"
            Type            = "Directory"
        }

        File Research 
        {
            Ensure          = "Present"
            DestinationPath = "c:\Share\Research"
            Type            = "Directory"
        }


        ### SHARING FOLDERS
		### Module : xSmbShare
		### https://www.powershellgallery.com/packages/xSmbShare/2.2.0.0

        xSmbShare Marketing 							
        { 
            Ensure       = "Present"  
            Name         = "Marketing" 
            Path         = "C:\Share\Marketing"  
            Description  = "Share Marketing" 
        } 

        xSmbShare HR 
        { 
            Ensure       = "Present"  
            Name         = "HR" 
            Path         = "C:\Share\HR"    
            Description  = "Share HR" 
        } 

        xSmbShare Production
        { 
            Ensure       = "Present"  
            Name         = "Production" 
            Path         = "C:\Share\Production"       
            Description  = "Share Production" 
        } 

        xSmbShare Logistics 
        { 
            Ensure       = "Present"  
            Name         = "Logistics" 
            Path         = "C:\Share\Logistics"      
            Description  = "Share Logistics" 
        } 

        xSmbShare Research
        { 
            Ensure       = "Present"  
            Name         = "Research" 
            Path         = "C:\Share\Research"              
            Description  = "Share Research" 
        } 


        ### IIS
		### Module : xPSDesiredStateConfiguration
        ### https://github.com/dsccommunity/xPSDesiredStateConfiguration

        WindowsFeature WebServer 
		{
			Ensure = "Present"
			Name   = "Web-Server"
		}

        WindowsFeature IISConsole 
        {
            Ensure    = 'Present'
            Name      = 'Web-Mgmt-Console'
            DependsOn = '[WindowsFeature]WebServer'
        }

        WindowsFeature IISScriptingTools 
        {
            Ensure    = 'Present'
            Name      = 'Web-Scripting-Tools'
            DependsOn = @('[WindowsFeature]WebServer', '[WindowsFeature]IISConsole')
        }
        
		File WebsiteContent 
		{
			Ensure          = 'Present'
            Type            = 'File'
			#SourcePath     = 'c:\test\index.htm'
			DestinationPath = 'c:\inetpub\wwwroot\index.htm'
            Contents        = 
            "<html>
                <body>
                    <h1>Beste Ken,</h1>
                    <br><h1>Deze website is nog in opbouw. Kom later terug a.u.b.!</h1>
                    <br><h1>Daniël</h1>
                </body>
            </html>"
		}
        

        ### FIREWALL FOR PORT 80 TCP/HTTP

        Firewall webservices                      
        {
            Name = "webservices"
            Ensure = "Present"
            Direction = "inbound"
            Description = "Allow traffic for webservices Port 80"
            Profile = "Domain"
            Protocol = "TCP"
            LocalPort = ("80")
            Action = "Allow"
            Enabled = "True"
        }
    }
}	


$cd = @{
    AllNodes = @(    
        @{  
            NodeName = "localhost"
            PsDscAllowPlainTextPassword = $true
        }
    ) 
}


BMOOS_Config -ConfigurationData $cd

Start-DscConfiguration .\BMOOS_Config -Verbose -Wait -Force
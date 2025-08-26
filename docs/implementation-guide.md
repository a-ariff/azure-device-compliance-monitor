# Implementation Guide

## Azure Device Compliance Monitor Setup

This comprehensive guide will walk you through setting up the Azure Device Compliance Monitor for real-time monitoring and automated remediation of Microsoft Intune and Azure AD managed devices.

## Prerequisites

### Required Permissions
- **Azure AD Permissions:**
  - Global Administrator or Intune Administrator role
  - Application Administrator (for app registration)
  - Cloud Device Administrator

- **Microsoft Graph API Permissions:**
  - `DeviceManagementManagedDevices.Read.All`
  - `DeviceManagementConfiguration.Read.All`
  - `Device.Read.All`
  - `Directory.Read.All`

### Technical Requirements
- PowerShell 5.1 or later
- Azure PowerShell module
- Microsoft Graph PowerShell SDK
- .NET Framework 4.7.2 or later

## Step 1: Azure App Registration

### 1.1 Create Application Registration
```powershell
# Connect to Azure AD
Connect-AzureAD

# Create new app registration
$app = New-AzureADApplication -DisplayName "Azure Device Compliance Monitor" \
    -HomePage "https://compliance.aglobaltec.com" \
    -ReplyUrls "https://compliance.aglobaltec.com/auth/callback"

# Create service principal
$sp = New-AzureADServicePrincipal -AppId $app.AppId
```

### 1.2 Configure API Permissions
```powershell
# Required Graph API permissions
$graphAPI = Get-AzureADServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"

# Add required permissions
Add-AzureADApplicationRequiredResourceAccess -ObjectId $app.ObjectId \
    -RequiredResourceAccess @{
        ResourceAppId = "00000003-0000-0000-c000-000000000000"
        ResourceAccess = @(
            @{ Id = "7ab1d382-f21e-4acd-a863-ba3e13f7da61"; Type = "Role" }, # Directory.Read.All
            @{ Id = "2f51be20-0bb4-4fed-bf7b-db946066c75e"; Type = "Role" }, # DeviceManagementManagedDevices.Read.All
            @{ Id = "dc377aa6-52d8-4e23-b271-2a7ae04cedf3"; Type = "Role" }  # DeviceManagementConfiguration.Read.All
        )
    }
```

### 1.3 Generate Client Secret
```powershell
# Create client secret
$secret = New-AzureADApplicationPasswordCredential -ObjectId $app.ObjectId \
    -CustomKeyIdentifier "ComplianceMonitor" \
    -EndDate (Get-Date).AddYears(2)

# Store these values securely
Write-Host "Application ID: $($app.AppId)"
Write-Host "Tenant ID: $((Get-AzureADTenantDetail).ObjectId)"
Write-Host "Client Secret: $($secret.Value)"
```

## Step 2: Environment Configuration

### 2.1 Configuration File Setup
Create `config/appsettings.json`:
```json
{
  "AzureAD": {
    "TenantId": "your-tenant-id",
    "ClientId": "your-client-id",
    "ClientSecret": "your-client-secret"
  },
  "Monitoring": {
    "PollingIntervalMinutes": 5,
    "MaxRetryAttempts": 3,
    "EnableAutomaticRemediation": false
  },
  "Notifications": {
    "EnableEmailAlerts": true,
    "EmailRecipients": ["admin@company.com"],
    "WebhookUrl": "https://your-webhook-url.com"
  }
}
```

### 2.2 Environment Variables
```powershell
# Set environment variables
$env:AZURE_TENANT_ID = "your-tenant-id"
$env:AZURE_CLIENT_ID = "your-client-id"
$env:AZURE_CLIENT_SECRET = "your-client-secret"
```

## Step 3: Script Deployment

### 3.1 Download and Extract
```powershell
# Clone the repository
git clone https://github.com/a-ariff/azure-device-compliance-monitor.git
cd azure-device-compliance-monitor

# Install required modules
Install-Module -Name Az -Force
Install-Module -Name Microsoft.Graph -Force
```

### 3.2 Initialize Monitoring
```powershell
# Import the monitoring module
Import-Module .\ComplianceMonitor.psm1

# Initialize compliance monitoring
Initialize-ComplianceMonitor -ConfigPath "config\appsettings.json"

# Start monitoring
Start-ComplianceMonitoring
```

## Step 4: Compliance Policies Configuration

### 4.1 Device Compliance Policies
Configure the following compliance policies in Intune:

- **Security Requirements:**
  - Encryption enabled
  - Screen lock configured
  - Antivirus protection active
  - OS version compliance

- **Application Requirements:**
  - Required apps installed
  - Prohibited apps not present
  - App protection policies applied

### 4.2 Conditional Access Integration
```powershell
# Configure conditional access policy
New-ConditionalAccessPolicy -Name "Device Compliance Required" \
    -State "Enabled" \
    -Conditions @{
        Applications = @{ IncludeApplications = "All" }
        Users = @{ IncludeUsers = "All" }
    } \
    -GrantControls @{
        RequireCompliantDevice = $true
        RequireMultiFactorAuthentication = $true
    }
```

## Step 5: Monitoring and Alerting

### 5.1 Dashboard Setup
The monitoring dashboard is available at:
- **Local Development:** `http://localhost:8080`
- **Production:** `https://compliance.aglobaltec.com`

### 5.2 Alert Configuration
```powershell
# Configure compliance alerts
Set-ComplianceAlert -Type "NonCompliantDevice" \
    -Threshold 5 \
    -NotificationMethod "Email" \
    -Recipients @("admin@company.com")

Set-ComplianceAlert -Type "PolicyViolation" \
    -Severity "High" \
    -AutoRemediate $true
```

## Step 6: Automated Remediation

### 6.1 Remediation Actions
Configure automatic remediation for common compliance issues:

```powershell
# Enable automatic remediation
Enable-AutoRemediation -PolicyType "PasswordComplexity" \
    -Action "NotifyUser" \
    -GracePeriodHours 24

Enable-AutoRemediation -PolicyType "AppProtection" \
    -Action "ReapplyPolicy" \
    -MaxAttempts 3
```

### 6.2 Custom Remediation Scripts
Create custom remediation scripts in the `remediation/` directory:

```powershell
# Example: Password policy remediation
function Invoke-PasswordPolicyRemediation {
    param(
        [string]$DeviceId,
        [string]$UserId
    )
    
    # Send notification to user
    Send-ComplianceNotification -UserId $UserId \
        -Message "Please update your password to meet company requirements"
    
    # Log remediation action
    Write-ComplianceLog -Level "Info" \
        -Message "Password policy remediation initiated for device $DeviceId"
}
```

## Step 7: Reporting and Analytics

### 7.1 Compliance Reports
Generate compliance reports:

```powershell
# Generate daily compliance report
Generate-ComplianceReport -Type "Daily" \
    -OutputPath "reports\daily-$(Get-Date -Format 'yyyy-MM-dd').xlsx"

# Generate executive summary
Generate-ComplianceReport -Type "Executive" \
    -Period "Monthly" \
    -EmailReport $true
```

### 7.2 Custom Analytics
Create custom analytics queries:

```powershell
# Query non-compliant devices by platform
$nonCompliantDevices = Get-ComplianceData | 
    Where-Object { $_.ComplianceState -eq "NonCompliant" } |
    Group-Object Platform |
    Select-Object Name, Count
```

## Step 8: Maintenance and Troubleshooting

### 8.1 Regular Maintenance
- **Weekly:** Review compliance reports
- **Monthly:** Update compliance policies
- **Quarterly:** Review and renew certificates

### 8.2 Troubleshooting
Common issues and solutions:

- **Authentication Failures:**
  ```powershell
  Test-AzureConnection -TenantId $env:AZURE_TENANT_ID
  ```

- **API Rate Limiting:**
  ```powershell
  Set-GraphThrottling -MaxRequestsPerMinute 50
  ```

- **Policy Sync Issues:**
  ```powershell
  Sync-CompliancePolicies -Force
  ```

## Security Considerations

### Data Protection
- Store credentials in Azure Key Vault
- Use managed identities where possible
- Implement least-privilege access
- Enable audit logging

### Network Security
- Use HTTPS for all communications
- Implement proper firewall rules
- Monitor for suspicious activities

## Support and Documentation

For additional support:
- Review the [troubleshooting guide](./troubleshooting.md)
- Check the [API reference](./api-reference.md)
- Visit the [project repository](https://github.com/a-ariff/azure-device-compliance-monitor)

---

*Last updated: August 2025*

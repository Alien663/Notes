# Encrypt Web.config (.netframework)

[![hackmd-github-sync-badge](https://hackmd.io/jSFo9hJiRdi4uRLi7xKmBA/badge)](https://hackmd.io/jSFo9hJiRdi4uRLi7xKmBA)

###### tags: `tutorials`

## Flow
1. server A 上產生出Mykey.xml
2. Mykey.xml匯入到 Server B上
3. 設定web.config
4. 並執行Encrypt指令(Debug model also can do it)
5. 設定權限給IIS App Pool，或是其他使用者

![](https://i.imgur.com/tBmN2lz.png)

## Test Project
> Config Path : D:\Test\AppConfig\TestConfigEncrypt\web.config

> IIS Website : TestEncryptConfig

### Web.config
```xml=
<configuration>
  <appSettings>
    <add key="webpages:Version" value="3.0.0.0" />
    <add key="webpages:Enabled" value="false" />
    <add key="ClientValidationEnabled" value="true" />
    <add key="UnobtrusiveJavaScriptEnabled" value="true" />
    <add key="Test" value="I believe I can fly" />
  </appSettings>
</configuration>
```

### Controller
```C#
using System;
using System.Web.Mvc;
using System.Configuration;

namespace TestConfigEncrypt.Controllers
{
    public class HomeController : Controller
    {
        public ActionResult Index()
        {
            ViewBag.Title = "Home Page";

            return View();
        }

        [HttpGet]
        [Obsolete]
        public ActionResult test()
        {
            var tt = ConfigurationSettings.AppSettings["test"];
            return Json(tt, JsonRequestBehavior.AllowGet);
        }
    }
}

```

## Script and Settings
### Server A
```powershell=
# go under the path of aspnet_regiis.exe
$Programs = @()
$ProgramName = 'aspnet_regiis.exe'
$SearchingPath = 'C:\Windows\Microsoft.NET\Framework'
$Programs += Get-ChildItem -Path $SearchingPath -Filter $ProgramName -Recurse -Force | %{$_.FullName}
cd $Programs[$Programs.count-1].replace('aspnet_regiis.exe', '')

# 1. server A 上產生出Mykey.xml
./aspnet_regiis.exe -pc "MyCustomKeys" –exp # create key
./aspnet_regiis.exe -px "MyCustomKeys" D:\MyCustomKeys.xml –pri # export key

# server A doen
```


### Server B
加密前，Debug模式下網站結果如下
![](https://i.imgur.com/9mSamYH.png)





首先，假設我們將Server A上的MyCustomKeys.xml也同樣複製到Server B的D槽中，也命名為MyCustomKeys.xml
```powershell=
# go under the path of aspnet_regiis.exe
$Programs = @()
$ProgramName = 'aspnet_regiis.exe'
$SearchingPath = 'C:\Windows\Microsoft.NET\Framework'
$Programs += Get-ChildItem -Path $SearchingPath -Filter $ProgramName -Recurse -Force | %{$_.FullName}
cd $Programs[$Programs.count-1].replace('aspnet_regiis.exe', '')

# 2. Mykey.xml匯入到 Server B上
./aspnet_regiis.exe -pi "MyCustomKeys" D:\MyCustomKeys.xml
```


為web.config加上設定，使其可以被加解密
之後指令會根據web.config裡面的Provider去尋找Key，並執行加解密
```xml=
<configProtectedData defaultProvider="MyProtectedProvider">
    <providers>
      <add name= "MyProtectedProvider"
           type= "System.Configuration.RsaProtectedConfigurationProvider, System.Configuration, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"
           keyContainerName= "MyCustomKeys"
           useMachineContainer="true" />
    </providers>
  </configProtectedData>
```

加密web.config，注意前面web.config新增的Provider名稱要對上，以及IIS App Pool也要給予權限
```powershell=
# 4. 並執行Encrypt指令，加密web.config
./aspnet_regiis.exe -pef appSettings "D:\Test\AppConfig\TestConfigEncrypt" -prov "MyProtectedProvider" 

# 5. 設定權限給IIS App Pool，或是其他使用者，假設網站命名為TestEncryptConfig
.\aspnet_regiis.exe -pa "MyCustomKeys" "IIS AppPool\TestEncryptConfig
```

web.config解密
```powershell=
./aspnet_regiis.exe -pdf appSettings "D:\Test\AppConfig\TestConfigEncrypt"
```
## Result
再次打開web.config，可以看到結果如下，已經是加密完的結果了
```xml=
<configuration>
    <appSettings configProtectionProvider="MyProtectedProvider">
        <EncryptedData Type="http://www.w3.org/2001/04/xmlenc#Element" xmlns="http://www.w3.org/2001/04/xmlenc#">
            <EncryptionMethod Algorithm="http://www.w3.org/2001/04/xmlenc#aes256-cbc" />
            <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
                <EncryptedKey xmlns="http://www.w3.org/2001/04/xmlenc#">
                    <EncryptionMethod Algorithm="http://www.w3.org/2001/04/xmlenc#rsa-oaep-mgf1p" />
                        <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
                            <KeyName>Rsa Key</KeyName>
                        </KeyInfo>
                    <CipherData>
                        <CipherValue>txNYOtN1Osl0l/l1iOKCzUn3czyPKe2GJL1QGjujaKLvYPgQdSeBYD7zFev19fFEhG9txK0DcTOn0v/h7eZbmvgVCwpKIQpXdif7FVg2xnG3LXO2Y2vKnc6yeYIUQ+3cX7xAew0JU521HsZJQN29aumviOKXxqConprmeRadFs/8C4wE+UjZkHtZJtuCE10FPwWAqMjmAv1SbIiy3wdnHJuOCH7Dey4wQ26xU6oL5WYFnHl5zw/+0BNkpJkQPUXvRgqbE8mTbThf4KI661Old8M9T0SIWbNABPRqiT6QOHbKo+8ERIvQHfQd3FSYKfnLKi0EREIUqc1Dxu53Tq2ePw==</CipherValue>
                    </CipherData>
                </EncryptedKey>
            </KeyInfo>
            <CipherData>
                <CipherValue>6k8KsU47ODGPJQJzNq5jTdkDLhlBNGf8sCoZr1hiXlMfYextfF3pzRzFHzAJ0Ec9oV7lGGuGHTP8bmTYs/Ak+DEo9CMRNZmpKzUG042mT/8CQKnRAJM6ZU1bxZMY5ZC1JBtm1dmFhN/SOZovvJBcD8fegK+U+M7o4WCge4AZ1dY35kZdjlytWE3BdsuwqJjn6WwhHTD+gvyLGsYe/uiYwrL0sN+7BjuuCDSPCZ69fbuS4OFd0eqNb5Un5r4jJTE3OcwD2FAMk5o0NCCcK7fLFzEzUm85MBLAomtyWeF4BQpgi4C93Faqiu5IrmwQrZewpdvgxKKHtITpgdZ4h1UVTz1/e/OljL4VP1pTHYeAJo39rscjXvXGtyQEySfDgp1llK6ZLSCluAnb1hh37/pnDCPUNQHhfzMVUnt/cvNcg6I=</CipherValue>
            </CipherData>
        </EncryptedData>
    </appSettings>
</configuration>
```

Debug模式下，網站運行結果依舊
![](https://i.imgur.com/5SwgWdv.png)


IIS架站情況下，如果沒有給予App Pool權限
![](https://i.imgur.com/xS3Dbtf.png)


給予App Pool權限後
![](https://i.imgur.com/R3Uy5Oz.png)



## My Code(deploy automatically)
```powershell=
# auto get server full name
$names = @("computername1", "computername2", "computername3")
$servers = @()
foreach( $name in $names){
    $servers += [System.Net.Dns]::GetHostByName($name).HostName
}

# send file from this server to others target server
Copy-Item "\\Remote\Folder\MyKey.xml" -Destination "D:\"
$FileContents = Get-Content -Path 'D:\MyKey.xml'
foreach( $server in $servers ){
    InVoke-Command -ComputerName $server -ScriptBlock{
        param($FilePath,$data)
        Set-Content -Path $FilePath -Value $data
    } -ArgumentList "D:\MyKey.xml",$FileContents
}

foreach( $server in $servers ){
    InVoke-Command -ComputerName $server -ScriptBlock{
        $myname = hostname
        echo "---------------------------------------------------------------------------------"
        echo "$myname start to run command"
        # auto change directiry of the aspnet_regiis.exe
        $Programs = @()
        $ProgramName = 'aspnet_regiis.exe'
        $SearchingPath = 'C:\Windows\Microsoft.NET\Framework'
        $Programs += Get-ChildItem -Path $SearchingPath -Filter $ProgramName -Recurse -Force | %{$_.FullName}
        cd $Programs[$Programs.count-1].replace('aspnet_regiis.exe', '')

        # import key
        .\aspnet_regiis.exe -pi "MykeyName" "D:\Mykey.xml"

        # add permission to user
        .\aspnet_regiis.exe -pa "MykeyName" "Domain\myAccount"

        # auto get IIS App Pool and set permission, some machine will error
        foreach($item in (Get-IISAppPool).Name){
            if(-not $item.Contains(".NET")){
                .\aspnet_regiis.exe -pa "MykeyName" "IIS AppPool\$item"
            }
        }

        # delete key from server
        Remove-Item D:\MyKey.xml

        # print done message
        $myname = hostname
        echo "$myname done"
        echo "---------------------------------------------------------------------------------"
    }
}
```
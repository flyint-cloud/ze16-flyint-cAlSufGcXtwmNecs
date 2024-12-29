
本文我们介绍针对Bios如何读取、写入数据，比如最常见的SN读取以及烧录。


在传统设备比如PC的工厂制造环节，需要完成数据初始化如SN、设备型号等，然后工厂测试流水线使用厂测软件验证。


还有一些数据需要存储在BIOS的需求，比如一些软件离线激活码，放在磁盘里肯定不合适，换个硬盘激活码就没了，那这种情况就可以将数据在BIOS存储及备份（注：最好是磁盘以及BIOS都存储，换硬盘、主板后均能重新激活）


### WMI查询


先看看WMI方式，可以用于查询和管理Windows系统的各种信息，包括读取BIOS信息


WMI\-Win32\_BIOS，可以查看Bios版本、制造商以及Bios Sn等：




```
 1             var searcher = new ManagementObjectSearcher("SELECT * FROM Win32_BIOS");
 2             Console.WriteLine("Win32_BIOS 获取bios信息:");
 3             foreach (var baseObject in searcher.Get())
 4             {
 5                 Console.WriteLine("Manufacturer: " + baseObject["Manufacturer"]);
 6                 Console.WriteLine("Version: " + baseObject["Version"]);
 7                 Console.WriteLine("SMBIOS BIOS Version: " + baseObject["SMBIOSBIOSVersion"]);
 8                 Console.WriteLine("Name: " + baseObject["Name"]);
 9                 Console.WriteLine("Release Date: " + baseObject["ReleaseDate"]);
10                 Console.WriteLine("Serial Number: " + baseObject["SerialNumber"]);
11                 Console.WriteLine("Status: " + baseObject["Status"]);
12                 Console.WriteLine("Caption: " + baseObject["Caption"]);
13                 Console.WriteLine("Build Number: " + baseObject["BuildNumber"]);
14                 Console.WriteLine("Code Set: " + baseObject["CodeSet"]);
15                 Console.WriteLine("Current Language: " + baseObject["CurrentLanguage"]);
16                 Console.WriteLine("Description: " + baseObject["Description"]);
17                 Console.WriteLine("Embedded Controller Major Version: " + baseObject["EmbeddedControllerMajorVersion"]);
18                 Console.WriteLine("Embedded Controller Minor Version: " + baseObject["EmbeddedControllerMinorVersion"]);
19                 Console.WriteLine("Identification Code: " + baseObject["IdentificationCode"]);
20                 Console.WriteLine("Installable Languages: " + baseObject["InstallableLanguages"]);
21                 Console.WriteLine("Install Date: " + baseObject["InstallDate"]);
22                 Console.WriteLine("Language Edition: " + baseObject["LanguageEdition"]);
23                 Console.WriteLine("List Of Languages: " + baseObject["ListOfLanguages"]);
24                 Console.WriteLine("Other Target OS: " + baseObject["OtherTargetOS"]);
25                 Console.WriteLine("Primary BIOS: " + baseObject["PrimaryBIOS"]);
26                 Console.WriteLine("System BIOS Major Version: " + baseObject["SystemBiosMajorVersion"]);
27                 Console.WriteLine("System BIOS Minor Version: " + baseObject["SystemBiosMinorVersion"]);
28                 Console.WriteLine("Target Operating System: " + baseObject["TargetOperatingSystem"]);29             }
30             searcher.Dispose();
```


也可以直接使用PowerShell：`Get-WmiObject -Class Win32_BIOS，以下是主要参数信息`



* SMBIOSBIOSVersion：BIOS版本号


* Manufacturer：BIOS制造商


* Name：BIOS名称


* ReleaseDate：BIOS发布的日期


* SerialNumber：系统序列号


* Version：BIOS版本



这是个人电脑获取结果：


![](https://img2024.cnblogs.com/blog/685541/202412/685541-20241217180512595-1314753679.png)


电脑\-系统信息中的Bios版本信息，是以上面Bios制造商\+Bios版本\+发布日期拼接显示：


![](https://img2024.cnblogs.com/blog/685541/202412/685541-20241217180923659-783294974.png)


WMI\-Win32\_BaseBoard，可以获取主板SN以及主板制造商：




```
 1             var searcher1 = new ManagementObjectSearcher("SELECT * FROM Win32_BaseBoard");
 2             Console.WriteLine("Win32_BaseBoard获取bios信息：");
 3             foreach (var baseObject in searcher1.Get())
 4             {
 5                 Console.WriteLine("Board Serial Number: " + baseObject["SerialNumber"]);
 6                 Console.WriteLine("Manufacturer: " + baseObject["Manufacturer"]);
 7                 Console.WriteLine("Product Name: " + baseObject["Product"]);
 8                 Console.WriteLine("Version: " + baseObject["Version"]);
 9             }
10             searcher1.Dispose();
```


也可以使用PowerShell查看，Get\-WmiObject \-Class Win32\_BaseBoard


个人电脑获取结果：


![](https://img2024.cnblogs.com/blog/685541/202412/685541-20241217182304029-1012191505.png)


另外，使用命令行工具，wmic bios get name, serialnumber, version也可以获取BIOS的基本信息


### Ami读写


获取Bios信息也可以使用AmiWin工具，同时此工具也可以用于数据写入。通过该工具可以轻松地自定义系统UUID、调整DMI信息，从而满足各种特定的系统需求比如指定BIOS位置的数据。其技术实现主要依赖于对BIOS固件的底层访问和修改


AMI全称American Megatrends Inc.，以BIOS/UEFI固件解决方案闻名。


读取指定位置数据：




```
 1     private static string ReadBiosData(string command)
 2     {
 3         var amiExePath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "AmiBios", "AMIDEWINx64.EXE");
 4         var startInfo = new ProcessStartInfo
 5         {
 6             FileName = $"\"{amiExePath}\"",
 7             Arguments = command,
 8             Verb = "runas",
 9             UseShellExecute = false,
10             RedirectStandardError = true,
11             RedirectStandardInput = true,
12             RedirectStandardOutput = true,
13             CreateNoWindow = true,
14             WindowStyle = ProcessWindowStyle.Hidden
15         };
16         using var process = Process.Start(startInfo);
17         if (process == null) return "";
18         process.WaitForExit();
19         string output = process.StandardOutput.ReadToEnd();
20 
21         if (string.IsNullOrWhiteSpace(output)) return "读取输出结果为空";
22         string[] inputs = output.Split(new[] { "\r\n" }, StringSplitOptions.RemoveEmptyEntries);
23         string targetOutput = inputs.FirstOrDefault(p => p.Contains(command));
24         if (targetOutput == null) return $"输出数据:{output}未找到指定Key[{command}]的目标输出";
25         string value = ExtractMiddleContent(targetOutput, "\"", "\"");
26         return value;
27     }
28     private static string ExtractMiddleContent(string source, string begin, string end)
29     {
30         var rg = new Regex("(?<=(" + begin + "))[.\\s\\S]*?(?=(" + end + "))", RegexOptions.Multiline | RegexOptions.Singleline);
31         return rg.Match(source).Value;
32     }
```


指定位置写入数据：




```
 1     private static string WriteBiosData(string command, string data)
 2     {
 3         var amiExePath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "AmiBios", "AMIDEWINx64.EXE");
 4         var args = $"{command} \"{data}\"";
 5         var startInfo = new ProcessStartInfo
 6         {
 7             FileName = $"\"{amiExePath}\"",
 8             Arguments = args,
 9             Verb = "runas",
10             UseShellExecute = false,
11             RedirectStandardError = true,
12             RedirectStandardInput = true,
13             RedirectStandardOutput = true,
14             CreateNoWindow = true,
15             WindowStyle = ProcessWindowStyle.Hidden
16         };
17         using var process = Process.Start(startInfo);
18         if (process == null) return "";
19         process.WaitForExit();
20         string output = process.StandardOutput.ReadToEnd();
21         if (string.IsNullOrWhiteSpace(output)) return "写入输出结果为空";
22         string[] inputs = output.Split(new[] { "\r\n" }, StringSplitOptions.RemoveEmptyEntries);
23         string targetOutput = inputs.FirstOrDefault(p => p.Contains(command));
24         if (targetOutput == null) return $"输出数据:{output}未找到指定指令[{command}]的目标输出";
25         return "成功";
26     }
```


下面使用AMI，对/BS位置测试读写：




```
 1     private void AmiReadButton_OnClick(object sender, RoutedEventArgs e)
 2     {
 3         var boardSn = ReadBiosData("/BS");
 4         Console.WriteLine($"读取/BS:{boardSn}");
 5 
 6         var result = WriteBiosData("/BS", "TEST");
 7         Console.WriteLine($"写入/BS:{result}");
 8         var boardSnNew = ReadBiosData("/BS");
 9         Console.WriteLine($"读取/BS:{boardSnNew}");
10     }
```


上方Demo见仓库：[https://github.com/kybs00/BiosDataDemo](https://github.com)


值得说明的是， Bios数据写入是一个高风险操作，操作时要避免断电，不然可能因为错误的BIOS文件导致一堆的启动问题


 


参考文章：


[AMIDEWINx64 v5\.27：BIOS修改与DMI编辑的利器\-CSDN博客](https://github.com):[楚门加速器](https://shexiangshi.org)



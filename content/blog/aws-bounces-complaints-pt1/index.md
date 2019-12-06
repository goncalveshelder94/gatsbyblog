---
title: "Handling Complaints"
date: 2019-10-30T15:07:02Z
img: "handling-complaints.jpg"
tags: [data]
author: "Helder Goncalves"
---

# Description
I'm in charge of handling a client's mailing list. We've recently sent out a batch of newsletters to which we had a couple subscriptors click `Spam` :sadface:

Since these people chose to mark the email as `Spam` instead of clicking the `unsubscribe` link on the newsletter, I'm tasked with removing them manually myself. The way I get ahold of these _complaint_ and _abuse_ emails is through an _S4_ bucket which is setup with _Amazon SES (Simple Email Service)_.

Opening the email files one by one, copying the recipient email address, heading back to the Django Admin interface, applying the email address as a filter and unsubscribing x 23... Some automation would be nice.

Keep reading to see what I've done.

# Application
1. How do I combine all of the files onto a single file? I could've just iterated through the entire directory with a for-loop but I chose to combine them and have a single file to worry about. PS: I'm programming in C#.

Since I'm also using Windows 10, recently moved away from my Hackintosh due to Catalina reasons, I googled up - "How to combine multiple files into a single one on Powershell" - I'm actually using the open source version _PowerShell Core_ [Github](https://github.com/PowerShell/PowerShell) - definitely worth a look.

Change directory to where the mail files are, which I've obtained through S3 CLI for Windows and:

```powershell
Get-Content * | Set-Content newfile.file
```   
Which means, get all content, pipe it through (imagine a virtual basket) and set the basket items onto a new basket `newfile.file`

**Success**
```powershell
>dir | findstr -I -N newfile
522:-a----        30/10/2019    11:14       11892431 newfile.file
```

**What does the file look like?**
```
This is an email abuse report for an email message from amazonses.com on Mon, 21 Oct 2019 15:34:36 +0000

------=_Part_4715xxxxxxx29770280
Content-Type: message/feedback-report
Content-Transfer-Encoding: 7bit
Content-Disposition: inline

Feedback-Type: abuse
User-Agent: Yahoo!-Mail-Feedback/2.0
Version: 0.1
Original-Mail-From: <01xxxxxxxxxxxxxxx2b0e-000000@eu-west-1.amazonses.com>
Original-Rcpt-To: johndoe@yahoo.co.uk
Received-Date: Mon, 21 Oct 2019 15:34:36 +0000
Reported-Domain: amazonses.com
Authentication-Results: authentication result string is not available
```
Along with another 7000-something occurences...

2. After some investigation I come to the conclusion that all I need is _Original-Rcpt-To: johndoe@yahoo.co.uk_

So I open _Visual Studio 2019_ (Community) and select the _.NET Core Console App_ in C# template.

```csharp
using System;
using System.IO;
using System.Text.RegularExpressions;
namespace MailExtractor
{
    class Program
    {
        static void Main(string[] args)
        {
            //Extract_lines("newfile.file", "filtered.file");
        }
    }
}
```

Afterwards I've googled - "Extract strings from file and save in another file in csharp" [Luckily, Google is talkative enough to get back to me with great results](https://stackoverflow.com/questions/11325028/extract-strings-from-file-and-save-in-another-using-c-sharp)

**Yes, StackOverflow to the rescue, I mean, why reinvent the wheel?**

And from there came the following snippet:
```csharp
private void extract_lines(string filein, string fileout)
    {
        using (StreamReader reader = new StreamReader(filein))
        {
            using (StreamWriter writer = new StreamWriter(fileout))
            {
                string line;
                while ((line = reader.ReadLine()) != null)
                {
                    if (line.Contains("what you looking for"))
                    {
                        writer.Write(line);
                    }
                }
            }
        }
    }
```
Which I modified to:

1. Look for a regular expression instead of using the `.Contains` method
2. Grabbed a regular expression from: I honestly can't remember where
3. Ended up at [https://haacked.com/archive/2007/08/21/i-knew-how-to-validate-an-email-address-until-i.aspx/](https://haacked.com/archive/2007/08/21/i-knew-how-to-validate-an-email-address-until-i.aspx/) where I made a final adjustment to the regexp. Regexps are so interesting and easy to master, I honestly encourage anyone to play with them if they haven't done so yet.
4. Through some VS Studio debugging, I realised how my new output was now an array (obviously, I've split it...) as you'll be able to see in the gif below. As such I've made the needed adjustments - ```writer.Write(newLine[1])``` since the content (email address) I needed was the object at index 1.

```csharp
public static void Extract_lines(string filein, string fileout)
    {
        Regex searchPattern = new Regex(@"Original-Rcpt-To: (?!\.)(""([^""\r\\]|\\[""\r\\])*""|"
            + @"([-a-z0-9!#$%&'*+/=?^_`{|}~]|(?<!\.)\.)*)(?<!\.)"
            + @"@[a-z0-9][\w\.-]*[a-z0-9]\.[a-z][a-z\.]*[a-z]$");
        // Regex searchPattern = new Regex(@"Original-Rcpt-To: \w+@[a-zA-Z_]+?\.[a-zA-Z]{2,3}$");
        using (StreamReader reader = new StreamReader(filein))
        {
            using (StreamWriter writer = new StreamWriter(fileout))
            {
                string line;
                string[] newLine;
                while ((line = reader.ReadLine()) != null)
                {
                    if (searchPattern.IsMatch(line))
                    {
                        newLine = line.Split(" ");
                        writer.WriteLine(newLine[1]);
                    }
                }
            }
        }
    }
```

The ```newLine = line.Split(" ");``` splits the line in the middle at "Original-Rcpt-To: " towards the email address and makes an array of two objects: 0: "Original-Rcpt-To:" and 1: "janedoe@yahoo.co.uk".

The ```writer.WriteLine(newLine[1]);``` where the ```writer``` is an instance of the class ```StreamWriter``` will write the object 1 to the new file for every line found by the regexp.

**The end result after running the program:**

![Filtered Emails](/images/filtered-content.jpg)

Stay tuned for part 2 where I take the emails and batch unsubscribe from a Django Newsletter application! :-)
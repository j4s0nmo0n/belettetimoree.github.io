## Abusing WSUS with MITM to perform ADCS ESC8 attack

In this article we will try to demonstrate another way (**not new**) to escalate privileges on a machine by abusing WSUS server misconfiguration.

Since 2020 when Gosecure Security teams released WSUS client update request [attack] path, and the famous tool pywsus, WSUS configurations are under increasing scrutiny during penetration tests. in order to compromise the client machine.
Then came Gosecure's [paper](https://www.gosecure.net/blog/2021/11/22/gosecure-investigates-abusing-windows-server-update-services-wsus-to-enable-ntlm-relaying-attacks/) about investigatng abusing Windows Server Update Services (WSUS) to enable NTLM Relaying Attacks.
While reading that great article, I noticed that Gosecure security teams let




Due to a plugin called `jekyll-titles-from-headings` which is supported by GitHub Pages by default. The above header (in the markdown file) will be automatically used as the pages title.

If the file does not start with a header, then the post title will be derived from the filename.

This is a sample blog post. You can talk about all sorts of fun things here.

---

### This is a header

#### Some T-SQL Code

```tsql
SELECT This, [Is], A, Code, Block -- Using SSMS style syntax highlighting
    , REVERSE('abc')
FROM dbo.SomeTable s
    CROSS JOIN dbo.OtherTable o;
```

#### Some PowerShell Code

```powershell
Write-Host "This is a powershell Code block";

# There are many other languages you can use, but the style has to be loaded first

ForEach ($thing in $things) {
    Write-Output "It highlights it using the GitHub style"
}
```

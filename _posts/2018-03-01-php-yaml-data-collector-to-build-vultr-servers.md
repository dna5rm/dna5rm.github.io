---
title: 'PHP YAML data collector to build Vultr servers'
date: '2018-03-01T00:00:34+00:00'
author: deaves
layout: post
permalink: /2018/03/php-yaml-data-collector-to-build-vultr-servers/
categories:
    - Ansible
    - Cloud
    - Linux
    - PHP
    - Vultr
tags:
    - ansible
    - array
    - automation
    - cloud
    - php
    - vultur
    - yaml
---

A few days ago I posted an [Ansible](https://www.ansible.com/) [playbook](/2018/02/ansible-provisioning-vultr-servers/) to provision &amp; deprovision virtual servers on [VULTR](https://www.vultr.com/?ref=7272485). One of the key features of that playbook would read a separate YAML file for its list of servers to build. The following is a simple yaml data collector written in PHP to generate the YAML for an Ansible playbook to consume.

![VR Server Request](/assets/phpform-vultr-req.png)

Long story short the heart of the script is the [yaml\_emit\_file](http://php.net/manual/en/function.yaml-emit-file.php) function that I use to dump POST data directly into a YAML file. As stated in my previous posting, anything could generate that file, be it the user themselves or a ticketing system.

## yaml-data-collector.php: PHP script, ASCII text

```php
<?php
/*
   # yaml-data-collector
   # Simple php form to collect information from a user on what servers to build.
   2017 (v1.0) - Script from www.davideaves.com
*/

if (basename($_SERVER["PHP_SELF"]) == basename(__FILE__) && isset($_GET["source"])) {
  header("Content-type: text/plain");
  exit(file_get_contents(basename($_SERVER["PHP_SELF"])));
}

// Uncomment to view debug points in the code.
//$debug = "yes";

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
  // Update $yamlfile
  yaml_emit_file("$_POST[tag].yaml", $_POST);
}

if(isset($_REQUEST['DATA'])):
  if (file_exists($_REQUEST['DATA'])) {
    $yamldata = yaml_parse_file($_REQUEST['DATA']);
    // Reindex the SERVERS array numerically incase broken.
    $yamldata['vultr']['servers'] = array_values($yamldata['vultr']['servers']);
  }
endif
?>
< !DOCTYPE html>
<html>
  <head>
     <title>< ?php echo gethostname(); ?>: VULTR Server Req.</title>
     <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
     <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate"/>
     <meta http-equiv="Pragma" content="no-cache"/>
     <script>
     /**
      * Projet: Dynamic dataTable.
      * Add/Delet rows from an html table.
      */

     var maxRows = 5;

     function addRow(tableID) {
          var table = document.getElementById(tableID);
          var rowCount = table.rows.length;
          if(rowCount < maxRows) {   // limit the user from creating fields more than your limits
               var row = table.insertRow(rowCount);
               var colCount = table.rows[0].cells.length;
               for(var i=0; i<colCount; i++) {
                    var newcell = row.insertCell(i);
                    newcell.innerHTML = table.rows[0].cells[i].innerHTML.replace(RegExp('\[[0-9]+\]'), '\[' + rowCount + '\]');
               }
          } else {
               alert("Maximum rows reached.");
          }
     }
     function deleteRow(tableID) {
          var table = document.getElementById(tableID);
          var rowCount = table.rows.length;
          for(var i=0; i<rowCount; i++) {
               var row = table.rows[i];
               var chkbox = row.cells[0].childNodes[0];
               if(null != chkbox && true == chkbox.checked) {
                    if(rowCount <= 1) {   // limit the user from removing all the fields
                         alert("Cannot remove all rows.");
                         break;
                    }
                    table.deleteRow(i);
                    rowCount--;
                    i--;
               }
          }
     }
     </script>
     <style>
     body {background-color: grey;}
     form {margin-top: 15px; margin-left: 15px; margin-right: 15px;}
     form a {color: blue; text-decoration: none; margin: 0px;}
     form label {display:inline-block; width:48%;}
     form input[type="text"] {width:48%;}
     form input[type="submit"] {width:150px;}
     form select {width:48%;}
     form .line {clear:both;}
     #DATANAV {background-color: #4ff; opacity: 0.5; border: solid 1px #999; border-radius: 5px; font-family: "Trebuchet MS", Arial, Helvetica, sans-serif; text-decoration: none; text-align: center; position: absolute; top: 15px; right: 15px; padding: 0 15px 0 15px; align: center;}
     #DATANAV a {text-decoration: none; font-size: 15px; color: red;}
     #DATANAV a:hover {font-weight: bold; color: maroon;}
     #DATANAV label {text-decoration: underline; color: black;}
     #TEMPLATE {width:1000px; margin:20px auto; background: #EEE; border: solid 1px #999; position: float; vertical-align: middle; border-radius: 5px;}
     #TEMPLATE fieldset {border-radius: 5px; border-width:2px; border-style:dotted; margin:5px;}
     #TEMPLATE h2 {margin:0; font-family:Arial, Sans-Serif; background:#5f9be3; color:#fff; white-space: nowrap; font-weight:bold; padding:0px; text-align: center;}
     #TEMPLATE span.config {background-color:black; color:white; font-family:monospace; white-space:pre; display:block;}
     #TEMPLATE mark {background-color:black; color:yellow; font-style:italic;}
     #TEMPLATE form {margin-top: 15px; margin-left: 15px; margin-right: 15px;}
     #TEMPLATE form label {display:inline-block; width:48%;}
     #TEMPLATE form a {color: blue; text-decoration: none; margin: 0px;}
     #TEMPLATE form input[type="text"] {width:50%;}
     #TEMPLATE form input[type="submit"] {width:100%;}
     #TEMPLATE form select {width:50.6%;}
     #TEMPLATE form .line {clear:both;}
     </style>
  </script></head>
<body>< ?php if (isset($debug)) {
  echo "<span class=\"debug\">";
  print_r(yaml_emit($_POST));
  echo "<hr /><br />\n";
} ?>

<div id="DATANAV">
<p><a class="button" href="?">New Request</a></p>
<p><label>Previous Tags</label>
<br />< ?php // SELECT THE TEMPLATE
  $templates = glob("*.yaml");
  foreach ($templates as $file) {
    echo "<a href='" . $_SERVER['PHP_SELF'] . "?DATA=" . $file . "'>" . preg_replace('/\\.[^.\\s]{3,4}$/', '', $file). "<br />\n";
  }
?></p></div>


<div id="TEMPLATE">
<form action="<?php echo $_SERVER['PHP_SELF']; ?>" method="POST" onsubmit="return confirm('Are you sure you want to submit this request?');">
<h2>Vultr Server Request</h2>
<fieldset class="row1">
 <legend>Request Details</legend>

 <p>
  <div class="line"><label>Tag</label>
   <input name="tag" type="text" required="required" value="<?php if(isset($yamldata['tag'])) { echo $yamldata['tag']; } ?/>" />
  </div>
  <div class="line"><label>Firewall Group</label>
   <select name="firewall_group">
    < ?php if(!isset($yamldata['firewall_group'])) { echo "<option value=\"\" disabled selected>Select your option"; } ?>
    <option value="shellserver" <?php if(isset($yamldata['firewall_group']) && $yamldata['firewall_group'] == "shellserver") { echo selected; } ?>>shellserver</option>
    <option value="webserver" <?php if(isset($yamldata['firewall_group']) && $yamldata['firewall_group'] == "webserver") { echo selected; } ?>>webserver</option>
   </select>
  </div>
 </p>

</fieldset>

<input type="hidden" name="modified" value="<?php echo date(DATE_RFC2822);?/>">

<fieldset class="row2">
 <legend>Server Details</legend>

  <p>
   <input type="button" value="Add Server" onClick="addRow('dataTable')" />
   <input type="button" value="Remove Server" onClick="deleteRow('dataTable')" />
  </p>

  <table id="dataTable" class="form">
   <tbody>< ?php if(isset($yamldata['vultr']['servers'])) { foreach($yamldata['vultr']['servers'] as $COUNT=>$SERVER){ ?><tr>
    <td><input type="checkbox" /></td>
    <td>
     <label>Domain Name</label>
     <input type="text" required="required" name="vultr[servers][<?php echo $COUNT; ?/>][name]" value="< ?php echo $SERVER['name']; ?>" />
    </td>
    <td>
     <label>Server OS</label>
     <select name="vultr[servers][<?php echo $COUNT; ?>][os]" required="required">
      < ?php if(!isset($yamldata['vultr']['servers'])) { echo "<option value=\"\" disabled selected>Select your option"; } ?>
      <option value="CentOS 7 x64" <?php if(isset($yamldata[vultr][servers][$COUNT][os]) && $yamldata[vultr][servers][$COUNT][os] == "CentOS 7 x64") { echo selected; } ?>>CentOS 7</option>
      <option value="Debian 9 x64 (stretch)" <?php if(isset($yamldata[vultr][servers][$COUNT][os]) && $yamldata[vultr][servers][$COUNT][os] == "Debian 9 x64 (stretch)") { echo selected; } ?>>Debian 9</option>
      <option value="Fedora 27 x64" <?php if(isset($yamldata[vultr][servers][$COUNT][os]) && $yamldata[vultr][servers][$COUNT][os] == "Fedora 27 x64") { echo selected; } ?>>Fedora 27</option>
      <option value="Ubuntu 17.10 x64" <?php if(isset($yamldata[vultr][servers][$COUNT][os]) && $yamldata[vultr][servers][$COUNT][os] == "Ubuntu 17.10 x64") { echo selected; } ?>>Ubuntu 17.10</option>
     </select>
    </td>
    <td>
     <label>Plan</label>
     <select name="vultr[servers][<?php echo $COUNT; ?>][plan]" required="required">
      < ?php if(!isset($yamldata['vultr']['servers'])) { echo "<option value=\"\" disabled selected>Select your option"; } ?>
      <option value="1024 MB RAM,25 GB SSD,1.00 TB BW" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['plan']) && $yamldata['vultr']['servers'][$COUNT]['plan'] == "1024 MB RAM,25 GB SSD,1.00 TB BW") { echo selected; } ?>>$5/month</option>
      <option value="2048 MB RAM,40 GB SSD,2.00 TB BW" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['plan']) && $yamldata['vultr']['servers'][$COUNT]['plan'] == "2048 MB RAM,40 GB SSD,2.00 TB BW") { echo selected; } ?>>$10/month</option>
      <option value="4096 MB RAM,60 GB SSD,3.00 TB BW" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['plan']) && $yamldata['vultr']['servers'][$COUNT]['plan'] == "4096 MB RAM,60 GB SSD,3.00 TB BW") { echo selected; } ?>>$20/month</option>
      <option value="8192 MB RAM,100 GB SSD,4.00 TB BW" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['plan']) && $yamldata['vultr']['servers'][$COUNT]['plan'] == "8192 MB RAM,100 GB SSD,4.00 TB BW") { echo selected; } ?>>$40/month</option>
      <option value="16384 MB RAM,200 GB SSD,5.00 TB BW" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['plan']) && $yamldata['vultr']['servers'][$COUNT]['plan'] == "16384 MB RAM,200 GB SSD,5.00 TB BW") { echo selected; } ?>>$80/month</option>
      <option value="32768 MB RAM,300 GB SSD,6.00 TB BW" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['plan']) && $yamldata['vultr']['servers'][$COUNT]['plan'] == "32768 MB RAM,300 GB SSD,6.00 TB BW") { echo selected; } ?>>$160/month</option>
     </select>
    </td>
    <td>
     <label>Region</label>
     <select name="vultr[servers][<?php echo $COUNT; ?>][region]" required="required">
      < ?php if(!isset($yamldata['vultr']['servers'])) { echo "<option value=\"\" disabled selected>Select your option"; } ?>
      <option value="Atlanta" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Atlanta") { echo selected; } ?>>Atlanta</option>
      <option value="Chicago" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Chicago") { echo selected; } ?>>Chicago</option>
      <option value="Dallas" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Dallas") { echo selected; } ?>>Dallas</option>
      <option value="Los Angeles" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Los Angeles") { echo selected; } ?>>Los Angeles</option>
      <option value="Miami" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Miami") { echo selected; } ?>>Miami</option>
      <option value="New Jersey" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "New Jersey") { echo selected; } ?>>New Jersey</option>
      <option value="Seattle" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Seattle") { echo selected; } ?>>Seattle</option>
      <option value="Silicon Valley" <?php if(isset($yamldata['vultr']['servers'][$COUNT]['region']) && $yamldata['vultr']['servers'][$COUNT]['region'] == "Silicon Valley") { echo selected; } ?>>Silicon Valley</option>
     </select>
    </td>
   </tr>< ?php } } else { ?><tr>
    <td><input type="checkbox" /></td>
    <td>
     <label>Domain Name</label>
     <input type="text" required="required" name="vultr[servers][0][name]" />
    </td>
    <td>
     <label>Server OS</label>
     <select name="vultr[servers][0][os]" required="required">
      < ?php if(!isset($yamldata['vultr[servers][0][os]'])) { echo "<option value=\"\" disabled selected>Select your option"; } ?>
      <option value="CentOS 7 x64">CentOS 7</option>
      <option value="Debian 9 x64 (stretch)">Debian 9</option>
      <option value="Fedora 27 x64">Fedora 27</option>
      <option value="Ubuntu 17.10 x64">Ubuntu 17.10</option>
     </select>
    </td>
    <td>
     <label>Plan</label>
     <select name="vultr[servers][0][plan]" required="required">
      <option value="" disabled selected>Select your option</option>
      <option value="1024 MB RAM,25 GB SSD,1.00 TB BW">$5/Month</option>
      <option value="2048 MB RAM,40 GB SSD,2.00 TB BW">$10/Month</option>
      <option value="4096 MB RAM,60 GB SSD,3.00 TB BW">$20/Month</option>
      <option value="8192 MB RAM,100 GB SSD,4.00 TB BW">$40/Month</option>
      <option value="16384 MB RAM,200 GB SSD,5.00 TB BW">$80/Month</option>
      <option value="32768 MB RAM,300 GB SSD,6.00 TB BW">$160/Month</option>
     </select>
    </td>
    <td>
     <label>Region</label>
     <select name="vultr[servers][0][region]" required="required">
      <option value="" disabled selected>Select your option</option>
      <option value="Atlanta">Atlanta</option>
      <option value="New Jersey">New Jersey</option>
      <option value="Chicago">Chicago</option>
      <option value="Dallas">Dallas</option>
      <option value="Los Angeles">Los Angeles</option>
      <option value="Miami">Miami</option>
      <option value="New Jersey">New Jersey</option>
      <option value="Seattle">Seattle</option>
      <option value="Silicon Valley">Silicon Valley</option>
     </select>
    </td>
   </tr>< ?php } ?></tbody>
  </table>

</fieldset>
<br />
<input class="submit" type="submit" value="Save Request &raquo;" />
</form><br />
</div>
</body>
</html>
```

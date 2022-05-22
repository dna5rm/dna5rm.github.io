---
title: 'YAML Templating Engine'
date: '2017-11-09T16:00:10+00:00'
author: deaves
layout: post
permalink: /2017/11/yaml-templating-engine/
categories:
    - Cisco
    - Firewall
    - Linux
    - 'Linux Scripts'
    - 'Load Balancing'
    - Networking
tags:
    - generator
    - noc
    - template
    - utilities
    - yaml
---

About a month ago I left my previous position and took a new engineering position. In my new role I am more focused on devops/orchestration for a new a network team. One major gap that I commonly see missing from most infrastructure/ops teams is the sharing of basic templates; ideally using a templating engine to maintain consistency. A couple years ago I stumbled upon a [templating-engine](https://github.com/mxtommy/templating-engine) by [mxtommy](https://github.com/mxtommy) on [github](https://github.com) that would read XML data and spit out a config. Aside from being a little rough, you would have to create your templates in a rough XML format plus it no longer seems to work with the latest version of PHP.

I decided that it was time to rewrite it; this time making it so it would read a YAML data input structure. YAML is a human-readable data serialization standard and is commonly used in orchestration tools like Ansible. Just like before this web script searches the current directory for template files (\*.yaml). It then parses the questions to ask and creates the appropriate html input fields. The user answers the questions and submits them back to the script. The script then replaces the user responses with the placeholders to create a constant and repeatable configuration… For example, last year I posted a config snippet for configuring a [Cisco UCS-E module on a 4000 series router](/2016/02/cisco-4000-series-isr-base-ucs-e-configuration/). A template created for that config would make for constant and rapid deployments of UCS-E blades by infrastructure staff. Additionally you could use it to create a bootstrap config to quickly bring a host online so Ansible can take over management of it.

This web script requires the YAML-1.1 parser and emitter for PHP…

- Debian: apt-get install php-yaml
- RHEL: yum install php-pecl-yaml

## template.php: Template Engine / PHP script.

```php
<?php
/*
   # yaml-templating-engine
 
   This web script searches the current directory for .yaml files. It will then ask
   questions based on questions defined in the template file, and then insert the info
   at the appropriate places in the template.
 
   Concept based on: https://github.com/mxtommy/templating-engine
 
   Rewritten to consume YAML and works with PHP7+.
 
   *** This is script not intended for use on a public server. ***
 
   2017 (v1.0) - Script from www.davideaves.com
*/
 
if (basename($_SERVER["PHP_SELF"]) == basename(__FILE__) && isset($_GET["source"])) {
  header("Content-type: text/plain");
  exit(file_get_contents(basename($_SERVER["PHP_SELF"])));
}
 
?>
< !DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8"/>
    <title>< ?php echo gethostname(); ?>: YAML Template Generator</title>
    <link href="template.css" rel="stylesheet"/>
  </head>
<body>
 
<table align="center"><tr>
	<td><h1>YAML Templating Engine</h1></td>
</tr></table>
 
<div id="TEMPLATE">
<fieldset>< ?php
 
// SELECT THE TEMPLATE
if (!isset($_REQUEST['template'])) {
  echo "<legend>Template Select\n";
  echo "<p>Please select one of the following templates:</p>\n<ul class=\"twocols\">\n";
  $templates = glob("*.yaml");
  foreach ($templates as $file) {
	print ("\t<li><a href='".$_SERVER['PHP_SELF']."?template=".$file."'>$file</a></li>\n");
  }
  echo "</ul>";
 
}
 
// ASK TEMPLATE QUESTIONS
elseif (!isset($_REQUEST['submitted'])) {
  echo "<legend>Template Input</legend>";
  echo "<h2>" . $_REQUEST['template'] . "</h2>\n";
 
  $yaml_template = yaml_parse_file($_REQUEST['template']);
  //print_r($yaml_template);
 
  echo "<form action=\"" . $_SERVER['PHP_SELF'] . "\" method=\"post\">\n";
  echo "<input type=\"hidden\" name=\"template\" value=\"" . $_REQUEST['template'] . "\"/>\n";	
 
  foreach($yaml_template['questions'] as $question) {
 
    // type is text
    if ($question['type'] == 'text') {
      echo "<div class=\"line\"><label>" . $question['description'] . ":</label><input type=\"text\" name=\"" . $question['name'] . "\" value=\"" . $question['default'] . "\"/></div>\n";
    }
 
    // type is select
    elseif($question['type'] == 'select') {
      echo "<div class=\"line\"><label>" . $question['description'] . ":</label><select name=\"" . $question['name'] . "\">\n";
      foreach ($question['option'] as $option) {
        echo "\t<option value=\"" . $option['name'] . "\"";
          if ($question['default'] == $option['name']) { echo " selected"; }
          echo ">" . $option['description'] . "</option>\n";
      }
      echo "</select></div>\n";
    }
 
    // type is checkbox
    elseif($question['type'] == 'checkbox') {
      echo "<div class=\"line\"><label>" . $question['description'] . ":</label><input type=\"checkbox\" name=\"" . $question['name'] . "\"";
      if ( isset($question['default']) ) { echo " checked"; }
      echo "/></div>\n";
    }
  }
  echo "<br /><input type=\"submit\" name=\"submitted\"/>\n</form>\n";
} 
 
// DISPLAY THE TEMPLATE
else {
  echo "<legend>Template Output</legend>";
  echo "<h2>" . $_REQUEST['template'] . "</h2>\n";
 
  $vars[] = "";
  $yaml_template = yaml_parse_file($_POST['template']);
 
  foreach ($yaml_template['questions'] as $i => $row) {
 
      $TMP = $yaml_template['questions'][$i];
 
      // Input type is text.
      if($TMP['type'] == "text") {
	$vars = $vars + array( "{{ ".$TMP['name']." }}" => "<mark>" . $_POST[$TMP['name']] . "</mark>" );
      }
 
      // Input type is select.
      elseif ($TMP['type'] == "select" ) {
        foreach ($TMP['option'] as $i => $row) {
          if($row['name'] == $_POST[$TMP['name']]) {
            foreach ($TMP['option'][$i]['option'] as $i => $row) {
              $vars = $vars + array( "{{ ".$row['name']." }}" => "<mark>" . $row['text'] . "</mark>" );
      } } } }
 
      // Input type is a checkbox.
      elseif ($TMP['type'] == "checkbox" ) {
        if(isset($_POST[$TMP['name']])) {
          $vars = $vars + array( "{{ ".$TMP['name']." }}" => "<mark>" . $TMP['checked'] . "</mark>" );
        } elseif (isset($TMP['unchecked'])) {
          $vars = $vars + array( "{{ ".$TMP['name']." }}" => "<mark>" . $TMP['unchecked'] . "</mark>" );
        } else {
          $vars = $vars + array( "{{ ".$TMP['name']." }}" => "" );
      } }
  }
 
  array_shift($vars);
  echo "<span class=\"config\">";
  echo strtr($yaml_template['config'], $vars);
  echo "</span>";
 
}
 
?></fieldset>
</div>
 
</body>
</html>
```

## template.css: Example CSS file for the template generator.

```css
/* TEMPLATE CSS */
#TEMPLATE {width:800px; margin:20px auto; background: #EEE; border: solid 1px #999; position: float; vertical-align: middle; border-radius: 5px;}
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
 
/* twocols layout */
.twocols {
  -webkit-column-count: 2; /* Chrome, Safari, Opera */
  -moz-column-count: 2; /* Firefox */
  column-count: 2;
}
```

Overall the templates are very easy to read and create making it so even non-technical support staff can contribute to the template repository this web script reads from.

## example.yaml: Example yaml template file.

```yaml
---
questions:

  - name: "hostname"
    description: "Hostname"
    type: "text"
    default: ROUTER

  - name: "mgmt_net"
    description: "Management Network"
    type: "select"
    default: dev
    option:
    - name: corp
      description: "Corporate"
      option:
      - name: VLAN
        text: 1000
      - name: VLAN-NAME
        text: PROD_VLAN
      - name: GATEWAY
        text: 192.168.50.1
    - name: dev
      description: "Development"
      option:
      - name: VLAN
        text: 2000
      - name: VLAN-NAME
        text: DEV_VLAN
      - name: GATEWAY
        text: 192.168.100.1

  - name: "service_encrypt"
    description: "Password Encryption"
    type: "checkbox"
    default: checked
    checked: service password-encryption
    unchecked: no service password-encryption

config: |
  !
  h0stname 
  !
  vlan 
   name 
  !
  ip default gateway
```

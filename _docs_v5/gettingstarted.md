---
layout: docs
title: Getting Started
---

Welcome to the COSMOS system... Let's get started! This guide is a high level overview that will help with setting up your first COSMOS project.

1. Get COSMOS Installed onto your computer by following the [Installation Guide](/docs/v5/installation).
   - You should now have COSMOS installed and a Demo project available that we can make changes to.
1. Browse to http://localhost:2900
   - The COSMOS Command and Telemetry Server will appear. This tool provides real-time information about each "target" in the system. Targets are external systems that receive commands and generate telemetry, often over ethernet or serial connections.
1. Experiment with other COSMOS tools. This is a DEMO environment so you can't break anything. Some things to try:
   - Use Command Sender to send individual commands.
   - Use Limits Monitor to watch for telemetry limits violations
   - Run some of the example scripts in Script Runner and Test Runner
   - View individual Telemetry packets in Packet Viewer
   - View detailed telemetry displays in Telemetry Viewer
   - Graph some data in Telemetry Grapher
   - View log type data in Data Viewer
   - Process log data with Data Extractor

## Interfacing with Your Hardware

Playing with the COSMOS Demo is fun and all, but now you want to talk to your own real hardware? Let's do it!

<div class="note info">
  <h5>Install and Platform</h5>
  <p>This guide assumes we're on Windows and COSMOS is installed in C:\COSMOS. Adjust scripts and paths as necessary to match your platform and COSMOS installation directory.</p>
</div>

1. Before creating your own configuration you should uninstall the COSMOS Demo so you're working with a clean COSMOS system. Click the Admin button and the PLUGINS tab. Then click the Trash can icon next to cosmos-demo to delete it. When you go back to the Command and Telemetry Server you should have a blank table with no interfaces.

1. Create a new "Configuration Directory". This directory is typically named after your program / project and for this example we'll call it "tutorial". Inside this directory it's recommended to create a README.md ([Markdown](https://www.markdownguide.org/)) to describe your program / project.

1. Now we need to create a plugin. Plugins are how we add targets and microservices to COSMOS. Our plugin will contain a single target which contains all the information defining the packets (command and telemetry) that are needed to communicate with the target. Use the COSMOS plugin generator to create the correct structure:

    ```batch
    C:\> cd tutorial
    C:\tutorial> C:\COSMOS\cosmos-control.bat cosmos generate plugin BOB
    ```

    This should create a new directory called "cosmos-bob" with a bunch of files in it. The full description of all the files is explained by the [Plugin Structure](/docs/v5/plugins#plugin-directory-structure) page.

1. The plugin generate creates a single target named after the plugin. Best practice is to create a single target per plugin to make it easier to share targets and upgrade them individually. Lets see what the plugin generate created for us. Open the cosmos-bob/targets/BOB/cmd_tlm/cmd.txt:

    ```bash
    COMMAND BOB EXAMPLE BIG_ENDIAN "Packet description"
      # Keyword           Name  BitSize Type   Min Max  Default  Description
      APPEND_ID_PARAMETER ID    16      INT    1   1    1        "Identifier"
      APPEND_PARAMETER    VALUE 32      FLOAT  0   10.5 2.5      "Value"
      APPEND_PARAMETER    BOOL  8       UINT   MIN MAX  0        "Boolean"
        STATE FALSE 0
        STATE TRUE 1
      APPEND_PARAMETER    LABEL 0       STRING          "COSMOS" "The label to apply"
    ```

    What does this all mean?
    - We created a COMMAND for target BOB named EXAMPLE.
    - The command is made up of BIG_ENDIAN parameters and is described by "Packet description". Here we are using the append flavor of defining parameters which stacks them back to back as it builds up the packet and you don't have to worry about defining the bit offset into the packet.
    - First we APPEND_ID_PARAMETER a parameter that is used to identify the packet called ID that is an 16-bit signed integer (INT) with a minimum value of 1, a maximum value of 1, and a default value of 1, that is described as the "Identifier".
    - Next we APPEND_PARAMETER a parameter called VALUE that is a 32-bit float (FLOAT) that has a minimum value of 0, a maximum value of 10.5, and a default value of 2.5.
    - Then we APPEND_PARAMETER a third parameter called BOOL which is a 8-bit unsigned integer (UINT) with a minimum value of MIN (meaning the smallest value a UINT supports, e.g 0), a maximum value of MAX (largest value a UINT supports, e.g. 255), and a default value of 0. BOOL has two states which are just a fancy way of giving meaning to the integer values 0 and 1. The STATE FALSE has a value of 0 and the STATE TRUE has a value of 1.
    - Finally we APPEND_PARAMETER called LABEL which is a 0-bit (meaning it takes up all the remaining space in the packet) string (STRING) with a default value of "COSMOS". Strings don't have minimum or maximum values as that doesn't make sense for STRING types.

    Check out the full [Command](/docs/v5/command) documention for more.

1. Now open the cosmos-bob/targets/BOB/cmd_tlm/tlm.txt:

    ```bash
    TELEMETRY BOB STATUS BIG_ENDIAN "Telemetry description"
      # Keyword      Name  BitSize Type   ID Description
      APPEND_ID_ITEM ID    16      INT    1  "Identifier"
      APPEND_ITEM    VALUE 32      FLOAT     "Value"
      APPEND_ITEM    BOOL  8       UINT      "Boolean"
        STATE FALSE 0
        STATE TRUE 1
      APPEND_ITEM    LABEL 0       STRING    "The label to apply"
    ```

    - This time we created a TELEMETRY packet for target BOB called STATUS that contains BIG_ENDIAN items and is described as "Telemetry description".
    - We start by defininig an ID_ITEM called ID that is a 16-bit signed integer (INT) with an id value of 1 and described as "Identifier". Id items are used to take unidentified blobs of bytes and determine which packet they are. In this case if a blob comes in with a value of 1, at bit offset 0 (since we APPEND this item first), interpreted as a 16-bit integer, then this packet will be "identified" as STATUS. Note the first packet defined without any ID_ITEMS is a "catch-all" packet that matches all incoming data (even if the data lengths don't match). 
    - Next we define three items similar to the command definition above.

    Check out the full [Telemetry](/docs/v5/telemetry) documention for more.    

1. COSMOS has defined an example command and telemetry packet for our target. Most targets will obviously have more than one command and telemetry packet. To add more simply create additional COMMAND and TELEMETRY lines in your text files. Actual packets should match the structure of your command and telemetry. Be sure to add at least one unique [ID_PARAMETER](/docs/v5/command#id_parameter) and [ID_ITEM](/docs/v5/telemetry#id_item) so your packets can be distinguished from each other.

1. Now we need to tell COSMOS how to connect to our BOB target. Open the cosmos-bob/plugin.txt file:

    ```bash
    # Set VARIABLEs here to allow variation in your plugin
    # See https://cosmosc2.com/docs/v5/plugins for more information
    VARIABLE bob_target_name BOB

    # Modify this according to your actual target connection
    # See https://cosmosc2.com/docs/v5/interfaces for more information
    TARGET BOB <%= bob_target_name %>
    INTERFACE <%= bob_target_name %>_INT tcpip_client_interface.rb 127.0.0.1 8080 8081 10.0 nil BURST
      MAP_TARGET <%= bob_target_name %>
    ```

    - This configures the plugin with a VARIABLE called bob_target_name with a default of "BOB". When you install this plugin you will have the option to change the name of this target to something other than "BOB". This is useful to avoid name conflicts and allows you to have multiple copies of the BOB target in your COSMOS system.
    - The TARGET line declares the new BOB target using the name from the variable. The <%= %> syntax is called ERB (embedded Ruby) and allows us to put variables into our text files, in this case referencing our bob_target_name.
    - The last line declares a new INTERFACE called (by default) BOB_INT that will connect as a TCP/IP client using the code in tcpip_client_interface.rb to address 127.0.0.1 (localhost) using port 8080 for writing and 8081 for reading. It also has a write timeout of 10 seconds and reads will never timeout (nil). The TCP/IP stream will be interpreted using the COSMOS [BURST](/docs/v5/protocols#burst-protocol) protocol which means it will read as much data as it can from the interface. For all the details on how to configure COSMOS interfaces please see the [Interface Guide](/docs/v5/interfaces). The MAP_TARGET line tells COSMOS that it will receive telemetry from and send commands to the BOB target using the BOB_INT interface.

<div class="note">
  <h5>Variables Support Reusability</h5>
  <p>In a plugin that you plan to reuse you should make things like hostnames and ports variables</p>
</div>

## Building Your Plugin

1. Now we need to build our plugin and upload it to COSMOS.

    ```batch
    C:\tutorial\cosmos-bob> C:\COSMOS\cosmos-control.bat cosmos rake build VERSION=1.0.0
      Successfully built RubyGem
      Name: cosmos-bob
      Version: 1.0.0.20210618174517
      File: cosmos-bob-1.0.0.20210618174517.gem
    ```

    - Note that the VERSION is required to specify the version to build. We recommend [sematic versioning](https://semver.org/) when building your plugin so people using your plugin (including you) know when there are breaking changes.

1. Once our plugin is built we need to upload it to COSMOS. Go back to the Admin page and click the Plugins Tab. Click on "Click to Select Plugin" and select the cosmos-bob-1.0.0.20210618174517.gem file. Then click Upload. Go back to the CmdTlmServer and you should see the plugin being deployed at which point the BOB_INT interface should appear and try to connect. Go ahead and click 'Cancel' because unless you really have something listening on port 8080 this will never connect. At this point you can explore the other CmdTlmServer tabs and other tools to see your newly defined BOB target.

1. Let's modify our BOB target and then update the copy in COSMOS. If you open Command Sender in COSMOS to BOB EXAMPLE you should see the VALUE parameter has value 2.5. Open the cosmos-bob/targets/BOB/cmd_tlm/cmd.txt and change the Default value for VALUE to 5 and the description to "New Value".

    ```bash
    COMMAND BOB EXAMPLE BIG_ENDIAN "Packet description"
      # Keyword           Name  BitSize Type   Min Max  Default  Description
      APPEND_ID_PARAMETER ID    16      INT    1   1    1        "Identifier"
      APPEND_PARAMETER    VALUE 32      FLOAT  0   10.5 5        "New Value"
      APPEND_PARAMETER    BOOL  8       UINT   MIN MAX  0        "Boolean"
        STATE FALSE 0
        STATE TRUE 1
      APPEND_PARAMETER    LABEL 0       STRING          "COSMOS" "The label to apply"
    ```

1. Rebuild the plugin with a new VERSION number. Since we didn't make any breaking changes we simply bump the patch release number:

    ```batch
    C:\tutorial\cosmos-bob> C:\COSMOS\cosmos-control.bat cosmos rake build VERSION=1.0.1
      Successfully built RubyGem
      Name: cosmos-bob
      Version: 1.0.1.20210618202504
      File: cosmos-bob-1.0.1.20210618202504.gem
    ```

1. Go back to the Admin page and click the Plugins Tab. This time click the clock icon next to cosmos-bob-1.0.0 to Upgrade the plugin. Browse to the newly built plugin gem and select it. This will re-prompt for the plugin variables (bob_target_name) so don't change the name and just click OK. You should see a message about the plugin being installed at which point the plugins list will change to cosmos-bob-1.0.1.20210618202504.gem. Go back to Command Sender and you should see the new Default value for VALUE is 5 and the description is "New Value". We have upgraded our plugin!

1. At this point you can create a new plugin named after your real target and start modifying the interface and command and telemetry definitions to enable COSMOS to connect to and drive your target. If you run into trouble look for solutions on our [Github Issues](https://github.com/BallAerospace/COSMOS/issues) page. If you would like to enquire about support contracts or professional COSMOS development please contact us at cosmos@ball.com.

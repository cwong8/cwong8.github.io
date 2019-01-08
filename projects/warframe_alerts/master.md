---
layout: page
title: Warframe Alerts visualization
permalink: /projects/warframe_alerts/
---

# Table of contents
1. [Introduction](#introduction)
2. [Obtaining the data through web scraping]({{ site.baseurl }}/projects/warframe_alerts/webscrape/)
3. [Tableau visualization](#tableau)
4. [Data](#data)

## Introduction <a name="introduction"></a>

Warframe is a free-to-play cooperative third-person shooter video game developed and published by Digital Extremes. In the game there are periodic timed alerts that players can complete for rewards including resources and weapon parts.

There is a twitter account that announces all alerts found [here.](https://twitter.com/WarframeAlerts)

## Tableau visualization <a name="tableau"></a>

<center>
    <div class='tableauPlaceholder' id='viz1546924205630' style='position: relative'><noscript><a href='#'><img alt=' ' src='https:&#47;&#47;public.tableau.com&#47;static&#47;images&#47;Wa&#47;WarframeAlerts&#47;Dashboard1&#47;1_rss.png' style='border: none' /></a></noscript><object class='tableauViz'  style='display:none;'><param name='host_url' value='https%3A%2F%2Fpublic.tableau.com%2F' /> <param name='embed_code_version' value='3' /> <param name='site_root' value='' /><param name='name' value='WarframeAlerts&#47;Dashboard1' /><param name='tabs' value='no' /><param name='toolbar' value='yes' /><param name='static_image' value='https:&#47;&#47;public.tableau.com&#47;static&#47;images&#47;Wa&#47;WarframeAlerts&#47;Dashboard1&#47;1.png' /> <param name='animate_transition' value='yes' /><param name='display_static_image' value='yes' /><param name='display_spinner' value='yes' /><param name='display_overlay' value='yes' /><param name='display_count' value='yes' /></object></div>                <script type='text/javascript'>                    var divElement = document.getElementById('viz1546924205630');                    var vizElement = divElement.getElementsByTagName('object')[0];                    vizElement.style.width='100%';vizElement.style.height=(divElement.offsetWidth*0.75)+'px';                    var scriptElement = document.createElement('script');                    scriptElement.src = 'https://public.tableau.com/javascripts/api/viz_v1.js';                    vizElement.parentNode.insertBefore(scriptElement, vizElement);                </script>
    <div class='tableauPlaceholder' id='viz1546924192357' style='position: relative'><noscript><a href='#'><img alt=' ' src='https:&#47;&#47;public.tableau.com&#47;static&#47;images&#47;Wa&#47;WarframeAlerts&#47;Dashboard2&#47;1_rss.png' style='border: none' /></a></noscript><object class='tableauViz'  style='display:none;'><param name='host_url' value='https%3A%2F%2Fpublic.tableau.com%2F' /> <param name='embed_code_version' value='3' /> <param name='site_root' value='' /><param name='name' value='WarframeAlerts&#47;Dashboard2' /><param name='tabs' value='no' /><param name='toolbar' value='yes' /><param name='static_image' value='https:&#47;&#47;public.tableau.com&#47;static&#47;images&#47;Wa&#47;WarframeAlerts&#47;Dashboard2&#47;1.png' /> <param name='animate_transition' value='yes' /><param name='display_static_image' value='yes' /><param name='display_spinner' value='yes' /><param name='display_overlay' value='yes' /><param name='display_count' value='yes' /><param name='filter' value='publish=yes' /></object></div>                <script type='text/javascript'>                    var divElement = document.getElementById('viz1546924192357');                    var vizElement = divElement.getElementsByTagName('object')[0];                    vizElement.style.width='100%';vizElement.style.height=(divElement.offsetWidth*0.75)+'px';                    var scriptElement = document.createElement('script');                    scriptElement.src = 'https://public.tableau.com/javascripts/api/viz_v1.js';                    vizElement.parentNode.insertBefore(scriptElement, vizElement);                </script>
</center>

## Data <a name="data"></a>

Data was taken from [RaidTime](https://raidtime.net/en/game/tool/warframe/alarm/pc/13e4970b99694f20cf7324335d207cbd) and covers alerts from 2018-10-24 16:35:01UTC to 2018-12-25 23:38:01UTC.

The data can be downloaded [here.]({{ site.baseurl }}/projects/warframe_alerts/data/alerts.csv)

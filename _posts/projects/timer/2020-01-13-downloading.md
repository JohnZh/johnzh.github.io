---
layout: default
title:  "Program Timer Downloading"
date:   2019-09-02 00:00:00 +0800
permalink: /programTimer/download
---

{% assign updateInfo = site.data.timer-update-info %}
<a id="download_url" href="{{ updateInfo.downloadUrl }}">Program Timer</a> is downloading...

<script type="text/javascript">
(function download() {
    document.getElementById('download_url').click();
})()
</script>
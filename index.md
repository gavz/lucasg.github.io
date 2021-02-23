---
layout: default
title: Home
---

{% assign publications = site.posts | concat: site.resources | sort: 'date' | reverse  %}


<table class="toctable">
<colgroup>
<col width="15%" />
<col width="70%" />
<col width="15%" />
</colgroup>
<thead>
<tr class="header">
<th>Date</th>
<th>URL</th>
<th>Type</th>
</tr>
</thead>
<tbody>
{% for resource in publications %}
<tr>
<td markdown="span"><time datetime="{{ resource.date | date_to_xmlschema }}" class="table-date">{{ resource.date | date_to_string }}</time></td>
{% if resource.external_url %}
	<td markdown="span"> <a href="{{ resource.external_url }}"> {{ resource.title }} </a> </td>
{% else %}
	<td markdown="span"> <a href="{{ resource.url }}"> {{ resource.title }} </a> </td>
{% endif %}

{% if resource.type %}
<td markdown="span"> {{ resource.type }}  </td>
{% else %}
<td markdown="span"> blog  </td>
{% endif %}

</tr>
{% endfor %}
</tbody>
</table> 

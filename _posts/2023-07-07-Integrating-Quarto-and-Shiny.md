# Integrating Quarto and Shiny
I have been working with quarto for a while, generating static web pages and pushing these to a shiny server. This has been fairly successful, however there is always this nagging feeling that I'll need to have a shiny app for some of these analysis projects. That day finally came in the form of a tabular display (coupled with downloads) that seemed best suited for a simple shiny app.

Perusing the internet, it appears there are official ways to integrate shiny code into your quarto site. I spent some time trying to get it to work but I couldn't. Perhaps it's the shiny server version that I'm running on.

I worked out a pretty simple approach that I'm documenting here for my own reference. If the official approach works for you, that's the better solution.

## A Solution
My solution was to create a quarto document that embeds the shiny app as an iframe. This keeps it in the website structure (as a quarto web page) but allows me to develop and deploy the shiny app.

In my quarto file:
````
```{=html}
<iframe width="1000" height="1000" src="Explorer" title="CMAP-RSI Explorer"></iframe>
```
````
I put this file in my main reports directory. Then, in a `Explorer` subdirectory is the shiny `app.R` file (and data file):
```
.:
CMAP-RSI_Explorer.qmd  Explorer  Overview.qmd  Radiosensitizers.qmd

./Explorer:
app.R  cmap.arrow
```

## Discussion
This works well with my workflow because I try to generate all of the data as a pipeline. So I can now generate `cmap.arrow` (the data file) as part of that process. The shiny app shows this data (it is a data.frame) and offers options for download (R, excel, csv, tsv). And thanks to the iframe, it's an addon to the reporting website rather than a completely separate thing (including the sidebar index, etc).


# Forcing Files into a Quarto Website
This is a small issue, but as part of a quarto website I was building for a project I wanted to include a shiny app (`app.R`) and a data file (`cmap.arrow`). I wanted to make this work with the `quarto render` approach. I used `rsync` instead of `quarto publish` to push the site to a shiny server.

It turns out that quarto has thought of many things, including this use case. In the `_quarto.yml` file that is used for details on the website you can include a clause for additional resources to bring along:

```yml
project:
    output-dir: public
    type: website
    resources:
        - "reports/Explorer/app.R"
        - "reports/Explorer/cmap.arrow"
```

With that small addition, `quarto render` will include these in the `public` directory during rendering. From here, I just `rsync` the `public` directory to the shiny server and I have both the shiny app and static webpages co-existing.

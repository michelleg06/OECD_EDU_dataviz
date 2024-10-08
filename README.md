# OECD Education and Skills `data-visuals`
A markdown file to show some data visuals using OECD Data on Education.

## Introduction
This repository hosts a couple of plots and maps made with the 🇷 package`ggplot2`. The visuals show current averages and trends on selected indicators. The document is meant to showcase the power 
💪 of publicly available data and, in the spirit of transparency, I'll include the data-wrangling and plotting code snippets 👩🏻‍💻. 

-----
## OECD Education and Skills
### Cloropleth of Investment in Education

the `R`packages used were: 

* {[ggplot2](https://ggplot2.tidyverse.org/)}: a system for declaratively creating graphics, based on [The Grammar of Graphics](https://www.amazon.com/Grammar-Graphics-Statistics-Computing/dp/0387245448/ref=as_li_ss_tl).
  
* {[cowplot](https://cran.r-project.org/web/packages/cowplot/vignettes/introduction.html)}: a simple add-on to ggplot. It provides various features that help with creating publication-quality figures, such as a set of themes, functions to align plots and arrange them into complex compound figures, and functions that make it easy to annotate plots and or mix plots with images.

* {[magick](https://docs.ropensci.org/magick/articles/intro.html): provide a modern and simple toolkit for image processing in R. It wraps the ImageMagick STL library.}

* {[maps](https://cran.r-project.org/web/packages/RColorBrewer/index.html)}: display of maps. Projection code and larger maps are in separate packages ('mapproj' and 'mapdata').

* {[scales](https://cran.r-project.org/web/packages/scales/index.html)}: a package that scales map data to aesthetics, and provides methods for automatically determining breaks and labels for axes and legends.

* {[ggtext](https://cran.r-project.org/web/packages/ggtext/index.html):} a `ggplot2` extension that enables the rendering of complex formatted plot labels (titles, subtitles, facet labels, axis labels, etc.). Text boxes with automatic word wrap are also supported.

* {[stringr](https://stringr.tidyverse.org/articles/from-base.html):} a consistent, simple and easy to use set of wrappers around the fantastic stringi package.

* {[cartography](https://cran.r-project.org/web/packages/cartography/vignettes/cartography.html):} offers thematic maps with the visual quality of those build with a classical mapping or GIS software. It includes color palettes for maps. 
  
* {[dplyr](https://stringr.tidyverse.org/articles/from-base.html):} dplyr is a grammar of data manipulation, providing a consistent set of verbs that help you solve the most common data manipulation challenges.

**The end result:**

![EDU_INVESTMENT_MAP](https://github.com/michelleg06/OECD_EDU_dataviz/blob/main/images/inv_edu_cloropleth2.png)

**A comment:**
Perhaps what I find most interest from the visualisation above, is that the countries that have invested the most in Education in the last years (as a percentage of their GDP) are in South America (Argentina, Brazil) and South Africa. These are not countries with the highest PISA outcomes. It made me think of the old adage "quality over quantity". An idea that has been extensively explored in the literature by Eric Hanushek (e.g., [2007](https://hanushek.stanford.edu/publications/education-quality-and-economic-growth)). It is not only enough to financially invest in education, we also need to mobilise the invested resources efficiently, and increase budget transparency and equitable spending. Almost 20 years later, this [World Bank Blog on Education Finance](https://www.worldbank.org/en/topic/education/brief/education-finance-using-money-effectively-is-critical-to-improving-education) still emphasises those points. 


**Code snippet:**

```r
oecd_invest_df <- read_csv("~/Desktop/OECD.EDU_investment.csv", )

# subset of relevant variables

oecd_invest_df <- oecd_invest_df[,c("REF_AREA", "Reference area", "TIME_PERIOD","OBS_VALUE")]
min(oecd_invest_df$OBS_VALUE, na.rm = T)
max(oecd_invest_df$OBS_VALUE, na.rm = T) # there is 1 value that makes no sense

subset(oecd_invest_df, `Reference area` == "South Africa") |>
  print() # south africa has one entry that doesn't match pattern

oecd_invest_df$OBS_VALUE <- ifelse(oecd_invest_df$OBS_VALUE > 1000, NA, oecd_invest_df$OBS_VALUE)

####### time trend for selected countries


# ---average expenditure as a percentage of GDP for all countries ----- #
# --------------------------------------------------------------------- #

world <- map_data('world')

# transform world into a simple features (sf) object
world_sf <- world |>
    st_as_sf(coords = c("long", "lat"), crs = 4326) %>% 
    #(CRS) number 4326 corresponds to the WGS 84 
    group_by(group, region) |> 
    # groups by group (mandatory) but also keeps region (needed to merge to other world/country files)
    summarize(geometry = st_combine(geometry)) |>
    st_cast("POLYGON")


# obtain a yearly average for each country

oecd_invest_df_avg <- oecd_invest_df |>
                            group_by(`Reference area`) |>
                            summarise(
                                average_investment_percGDP = mean(OBS_VALUE, na.rm = TRUE)
                            ) |>
                            drop_na() |>
                            rename(region = `Reference area`)

# which country names are not the same?
idx_oecd  <- unique(oecd_invest_df$`Reference area`) #52
idx_world <- unique(world_sf$region) #252

common_idx <- intersect(idx_oecd,idx_world)
length(common_idx) # 42... #

idx_oecd_butnotinWorld_diff <- setdiff(idx_oecd,idx_world)

cat("Countries in OECD data but not in World df: ", idx_oecd_butnotinWorld_diff, "\n")

# -: How many can we find in World with different spelling?
unique(world_sf$region)

# ---- string cleaning ----- #
# -------------------------- #
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "United Kingdom", "UK")
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "Czechia", "Czech Republic")
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "Türkiye", "Turkey")
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "China (People’s Republic of)", "China")
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "United States", "USA")
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "Slovak Republic", "Slovakia")
oecd_invest_df_avg$region <- str_replace(oecd_invest_df_avg$region, "Korea", "South Korea")

# -------------------------- #
# merge data subset for  with the lat and long sf dataset
oecd_sf <-  world_sf|>
                left_join(oecd_invest_df_avg, by ="region") |>
                st_as_sf()

# use ggplot to draw the map with geom_sf() :)
colors <- carto.pal(pal1 = "blue.pal", n1=5) # n1 = 5 'n' is the number of breaks to be displayed in the legend


map_edu_exp_percGDP <- oecd_sf |>
  ggplot() +
  geom_sf(aes(fill = average_investment_percGDP)) + 
  scale_fill_gradientn(
    colors = colors,  # Continuous color scale
    labels = scales::percent_format(scale = 1,)
  ) + 
  theme_minimal() + 
  labs(
    #title = "Investment in Education as a % of GDP",
    subtitle = "Data from the <span style='color:cornflowerblue'>OECD Data Explorer</span>:\n Expenditure on educational institutions as a percentage of GDP, from primary to tertiary level. Estimated average investment between the period 2015-2020.",
    fill = NULL  # Remove legend title
  ) +  
  theme(
    axis.line = element_blank(),
    axis.text.x = element_blank(),
    axis.text.y = element_blank(),
    axis.ticks = element_blank(),
    axis.title.x = element_text(size = 9, color = "grey60", hjust = 0, vjust = 10),
    axis.title.y = element_blank(),
    legend.position = "right",  # Position the legend on the right
    legend.text = element_text(size = 9, color = "grey20"),
    legend.key.height = unit(2, "cm"),  # Adjust the height of the legend keys
    panel.grid.major = element_line(color = "white", size = 0.4),
    panel.grid.minor = element_blank(),
    plot.margin = unit(c(t = 0, r = 0, b = 2.5, l = 0), "lines"),
    plot.title = element_text(face = "bold", size = 20, color = "black", hjust = 0.5),
    plot.subtitle = element_textbox_simple(size = 10, color = "grey60", hjust = 0.5, 
                                           lineheight = 1.2,  # Ensure proper spacing
                                           padding = margin(5, 5, 5, 5),  # Optional padding
                                           box.color = "white"),  # Background color (optional)
    plot.background = element_rect(fill = "white", color = NA), 
    panel.background = element_rect(fill = "white", color = NA), 
    legend.background = element_rect(fill = "white", color = NA),
    panel.border = element_blank()
  )

print(map_edu_exp_percGDP)

# add the oecd logo
oecd_logo <- image_read("~/Desktop/oecd_logo.jpeg")

title_with_logo <- cowplot::ggdraw() +
    draw_image(oecd_logo, x = 0.01, y = 0.1, width = 0.1) +
    draw_label("Investment in Education as a % of GDP", x = 0.12, y = 0.55, 
               hjust = 0, vjust = 0.5, fontface = "bold", size = 18) #

# Combine the title with the bar plot using cowplot's plot_grid()
map_w_logo <- plot_grid(
    title_with_logo,
    map_edu_exp_percGDP,
    ncol = 1,
    rel_heights = c(0.1, 1)  # adjust the relative heights of the title and the map 
)

print(map_w_logo)

ggsave("inv_edu_cloropleth.pdf", map_w_logo)

```
-----
### Lineplots Scatterplots: Exploring the various dimensions of the PISA dataset

the `R`packages used were: 

* {[ggplot2](https://ggplot2.tidyverse.org/)}: a system for declaratively creating graphics, based on [The Grammar of Graphics](https://www.amazon.com/Grammar-Graphics-Statistics-Computing/dp/0387245448/ref=as_li_ss_tl).

* {[dplyr](https://stringr.tidyverse.org/articles/from-base.html):} dplyr is a grammar of data manipulation, providing a consistent set of verbs that help you solve the most common data manipulation challenges.

* {[learningtower](https://cran.r-project.org/web/packages/learningtower/index.html):} OECD PISA Datasets from 2000-2018 in an Easy-to-Use Format.

* {[ggtext](https://cran.r-project.org/web/packages/ggtext/index.html):} A 'ggplot2' extension that enables the rendering of complex formatted plot labels.

* {[ggrepel](https://ggrepel.slowkow.com/):} provides geoms for ggplot2 to repel overlapping text labels.

**The graphs:**

![lineplot](https://github.com/michelleg06/OECD_EDU_dataviz/blob/main/images/p.png)

**A comment:**
Lineplots are a great way to visualise time trends. For the above plot, I selected 6 OECD countries at random, two of which are considered emerging markets and the rest developed economies. There are two very stark trends. First, the maths performance gap between emerging and developed economies is slowly decreasing, but remains salient. Second, whilst students from emerging economies seem to be improving over time, the opposite has happened for students in advanced economies! 

What else could we get from these data?

![scatterplots](https://github.com/michelleg06/OECD_EDU_dataviz/blob/main/images/trends.png)

**A comment:**
The scatterplots above show a stark correlation between students' performance in maths and their economic, social, and cultural status. In turn, a student's ESCS index is correlated to the development status of a country. So, guaranteeing the well-being of a student can significantly improve their learning capacity. This association is not a new finding, many studies have looked at it from a correlational and even causal lens (e.g. [Yakovlev and  Leguizamon (2012](https://www.sciencedirect.com/science/article/pii/S1053535712001035?casa_token=hr8QCOmUWxIAAAAA:AESVTCc2y-UJZr1H_zvi1mWrQWMLoDBkheMxCvyjUjWdcvFHo0uyg9b7vy5O8KL5Kw_q-DMyXpI)); nonetheless, the composite index measured by the OECD is one of the first that attempts to understand the relationship between multidimensional well-being and education (since it is collected under the PISA assessment). An interesting note on the scatterplot on Panel B: by replacing the geom dot for the year label, we can get an impression of what we more easily observed with the line plot: the best (maths) performing years for developed economies are in the past, whereas for emerging economies' students, their best performing years are more recent. 


**Code snippet**

```r
# student data
student_data_all <- load_student("all")


student_subset <- student_data_all |>
                            group_by(year,country) |>
                            summarize(
    mean_math = mean(math, na.rm = TRUE),
    mean_read = mean(read, na.rm = TRUE),
    mean_escs = mean(escs, na.rm = TRUE)
  )

# selected countries

student_subset <- subset(student_subset, student_subset$country=="BRA"|student_subset$country=="CAN" | student_subset$country=="MEX" | student_subset$country=="FRA" | student_subset$country=="GBR"| student_subset$country=="ESP")

# line plot

p <- student_subset |>
  ggplot() +
  geom_line(aes(x = year, y = mean_math, color = country, group = country)) +
  labs(
    title = "Performance in Mathematics PISA assessment",
    subtitle = "Data was obtained from the <span style='font-family:Courier; background-color:#f0f0f0; padding:2px 4px; border-radius:3px; border:1px solid #d3d3d3;'>learningtower</span> R package which provides a clean panel dataset from the <span style='color:cornflowerblue'>OECD's PISA international student assessment</span>. Graph shows pooled results between the years 2000 and 2018."
  ) +
  theme_minimal() +  # Place theme_minimal() before custom theme adjustments
  theme(
    plot.title = element_text(size = 16, face = "bold"),
    plot.subtitle = ggtext::element_textbox_simple(size = 10, color = "gray")
  ) +
  labs(
      x = "Year",      
      y = "Mathematics Score (country average)",             
      colour = "Country Name"
  )


print(p)
ggsave("p.png", p)



# # # relationship with socioeconomic status


p2 <- student_subset |>
            ggplot(aes(x = mean_math, y = mean_escs, colour = country)) +
            geom_point() +
            theme_minimal() +
            labs(
                x = "Mathematics Score (country average)",      
                y = "ESCS Score (country average)",             
                colour = "Country Name"            
            )
        
print(p2)
ggsave("math_escs.png", p2)


p2.1 <- student_subset |>
    ggplot(aes(x = mean_math, y = mean_escs, colour = country, label = year)) +
    geom_text_repel(size = 3, show.legend = FALSE) +  # Avoid showing legend for text
    geom_point(aes(colour = country), size = 0, shape = 16, show.legend = TRUE) +  # Add invisible points for legend
    theme_minimal() +
    guides(
        colour = guide_legend(
            override.aes = list(label = NULL, size = 5, shape = 16)
        )
    ) +
    labs(
                x = "Mathematics Score (country average)",      
                y = "ESCS Score (country average)",             
                colour = "Country Name"            
            )

print(p2.1)

grid.arrange(p2, p2.1)

ggsave("maths_yearlegend.png",p2.1)


```


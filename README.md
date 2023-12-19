# 1_Shot_Chart_Tutorial.Rmd
---
title: "1_Shot_Chart_Tutorial"
output: html_notebook
---

# Packages
```{r}
library(ggplot2)
library(tidyverse)
library(nbastatR)
library(devtools)
library(ncaahoopR)
library(extrafont)
library(cowplot)
```
# Creating Court 
```{r}
# Creating court and plotting

circle_points = function(center = c(0, 0), radius = 1, npoints = 360) {
  angles = seq(0, 2 * pi, length.out = npoints)
  return(data_frame(x = center[1] + radius * cos(angles),
                    y = center[2] + radius * sin(angles)))
}

# Court Dimenons & lines
width = 50
height = 94 / 2
key_height = 19
inner_key_width = 12
outer_key_width = 16
backboard_width = 6
backboard_offset = 4
neck_length = 0.5
hoop_radius = 0.75
hoop_center_y = backboard_offset + neck_length + hoop_radius
three_point_radius = 23.75
three_point_side_radius = 22
three_point_side_height = 14

# Court themes
court_themes = list(
  light = list(
    court = 'floralwhite',
    lines = '#999999',
    text = '#222222',
    made = '#00bfc4',
    missed = '#f8766d',
    hex_border_size = 1,
    hex_border_color = "#000000"
  ),
  dark = list(
    court = '#000004',
    lines = '#999999',
    text = '#f0f0f0',
    made = '#00bfc4',
    missed = '#f8766d',
    hex_border_size = 0,
    hex_border_color = "#000000"
  ),
  ppt = list(
    court = 'gray20',
    lines = 'white',
    text = '#f0f0f0',
    made = '#00bfc4',
    missed = '#f8766d',
    hex_border_size = 0,
    hex_border_color = "gray20"
)
)

# Function to create court based on given dimensions
plot_court = function(court_theme = court_themes$light, use_short_three = FALSE) {
  if (use_short_three) {
    three_point_radius = 22
    three_point_side_height = 0
  }
  
  court_points = data_frame(
    x = c(width / 2, width / 2, -width / 2, -width / 2, width / 2),
    y = c(height, 0, 0, height, height),
    desc = "perimeter"
  )
  
  court_points = bind_rows(court_points , data_frame(
    x = c(outer_key_width / 2, outer_key_width / 2, -outer_key_width / 2, -outer_key_width / 2),
    y = c(0, key_height, key_height, 0),
    desc = "outer_key"
  ))
  
  court_points = bind_rows(court_points , data_frame(
    x = c(-backboard_width / 2, backboard_width / 2),
    y = c(backboard_offset, backboard_offset),
    desc = "backboard"
  ))
  
  court_points = bind_rows(court_points , data_frame(
    x = c(0, 0), y = c(backboard_offset, backboard_offset + neck_length), desc = "neck"
  ))
  
  foul_circle = circle_points(center = c(0, key_height), radius = inner_key_width / 2)
  
  foul_circle_top = filter(foul_circle, y > key_height) %>%
    mutate(desc = "foul_circle_top")
  
  foul_circle_bottom = filter(foul_circle, y < key_height) %>%
    mutate(
      angle = atan((y - key_height) / x) * 180 / pi,
      angle_group = floor((angle - 5.625) / 11.25),
      desc = paste0("foul_circle_bottom_", angle_group)
    ) %>%
    filter(angle_group %% 2 == 0) %>%
    select(x, y, desc)
  
  hoop = circle_points(center = c(0, hoop_center_y), radius = hoop_radius) %>%
    mutate(desc = "hoop")
  
  restricted = circle_points(center = c(0, hoop_center_y), radius = 4) %>%
    filter(y >= hoop_center_y) %>%
    mutate(desc = "restricted")
  
  three_point_circle = circle_points(center = c(0, hoop_center_y), radius = three_point_radius) %>%
    filter(y >= three_point_side_height, y >= hoop_center_y)
  
  three_point_line = data_frame(
    x = c(three_point_side_radius, three_point_side_radius, three_point_circle$x, -three_point_side_radius, -three_point_side_radius),
    y = c(0, three_point_side_height, three_point_circle$y, three_point_side_height, 0),
    desc = "three_point_line"
  )
  
  court_points = bind_rows(
    court_points,
    foul_circle_top,
    foul_circle_bottom,
    hoop,
    restricted,
    three_point_line
  )
  
  
  court_points <- court_points
  
  # Final plot creation
  ggplot() +
    geom_path(
      data = court_points,
      aes(x = x, y = y, group = desc),
      color = court_theme$lines
    ) +
    coord_fixed(ylim = c(0, 45), xlim = c(-25, 25)) +
    theme_minimal(base_size = 22) +
    theme(
      text = element_text(color = court_theme$text),
      plot.background = element_rect(fill = 'gray20', color = 'gray20'),
      panel.background = element_rect(fill = court_theme$court, color = court_theme$court),
      panel.grid = element_blank(),
      panel.border = element_blank(),
      axis.text = element_blank(),
      axis.title = element_blank(),
      axis.ticks = element_blank(),
      legend.background = element_rect(fill = court_theme$court, color = court_theme$court),
      legend.margin = margin(-1, 0, 0, 0, unit = "lines"),
      legend.position = "bottom",
      legend.key = element_blank(),
      legend.text = element_text(size = rel(1.0))
    )
}
```

# NBA Data
```{r}
Sys.setenv("VROOM_CONNECTION_SIZE" = "262144")  # Set to 256 KB

# Grab team shot data
nets <- teams_shots(teams = "Brooklyn Nets", seasons = 2021, season_types = "Playoffs") # Get team szn shot charts
# Using location X and location Y

# Filter shot data for player & clean data to fit court dimensions
durant <- nets %>% filter(namePlayer=="Kevin Durant") %>% 
  mutate(x = as.numeric(as.character(locationX)) / 10, y = as.numeric(as.character(locationY)) / 10 + hoop_center_y)

# Horizontally flip the data
durant$x <- durant$x * -1 

# Filter shots by game date
final_durant <- durant %>% filter(dateGame == 20210615) #filter to one game 
```

# NBA Chart
```{r}
p1 <- plot_court(court_themes$ppt, use_short_three = F) + #Not a short three-point line since its nba 
  geom_point(data = final_durant, aes(x = x, y = y, color = final_durant$isShotMade, fill = final_durant$isShotMade), # x = xlocation and y = ylocation of shots
             size =3, shape = 21, stroke = .5) +  
  scale_color_manual(values = c("green4","red3"), aesthetics = "color", breaks=c("TRUE", "FALSE"), labels=c("Made", "Missed")) +
  scale_fill_manual(values = c("green2","gray20"), aesthetics = "fill", breaks=c("TRUE", "FALSE"), labels=c("Made", "Missed")) +
  scale_x_continuous(limits = c(-27.5, 27.5)) +
  scale_y_continuous(limits = c(0, 45)) +
  theme(plot.title = element_text(hjust = .5, size = 22, family = "Comic Sans MS", face = "bold", vjust = -4), # hjust = how centrered is the title, 0-1 scale; vjust = moving text vertical
        plot.subtitle = element_text(hjust = .5, size = 10, family = "Comic Sans MS", face = "bold", vjust = -8),
        legend.position = c(.5, .85),
        legend.direction = "horizontal",
        legend.title = element_blank(),
        legend.text = element_text(hjust = .5, size = 10, family = "Comic Sans MS", face = "bold", colour = "white"),
        plot.caption = element_text(hjust = .5, size = 6, family = "Comic Sans MS", face = "bold", colour = "lightgrey", vjust = 8)) +
  ggtitle(label = "Kevin Durant vs. Milwaukee",
          subtitle = "49 PTS | 17 REB | 10 AST | 4-9 3PT - 6/15/21") +
  labs(caption = "Tutorial: @DSamangy")

ggdraw(p1) + theme(plot.background = element_rect(fill="gray20", color = NA)) 

ggsave("Durant.png", height = 6, width = 6, dpi = 300)
```
![Durant](https://github.com/jjsilverman9/1_Shot_Chart_Tutorial.Rmd/assets/149702273/dc358cbb-63ac-4c39-a5b2-22c557ea9ffb)

# NCAA Data
```{r}
# Get Schedule w/ game IDS
cuse_schedule <- get_schedule("Syracuse", season = "2020-21")

# Grab game ids of all 2021 Cuse games
cuse_ids <- cuse_schedule$game_id[1:28] #all games froms cuse_schedule

# Grab all shots from Cuse games 
cuse_shots <- get_shot_locs(cuse_ids) #get shot locations from both teams from every cuse game that year

# Filter shot data by player & game
buddy <- cuse_shots %>% filter(shooter=="Buddy Boeheim", game_id==401310900)

# Change data to fit the court dimensions
buddy <- buddy %>% mutate(x=x-25)
#buddy <- buddy %>% mutate(y = 94-y)
```

# NCAA Chart
```{r}
p1 <- plot_court(court_themes$ppt, use_short_three = T) +
  geom_point(data = buddy, aes(x = x, y = y, color = buddy$outcome, fill = buddy$outcome), 
             size =3, shape = 21, stroke = .5) +  
  scale_color_manual(values = c("green4","red3"), aesthetics = "color", labels=c("Made", "Missed")) +
  scale_fill_manual(values = c("green2","gray20"), aesthetics = "fill", labels=c("Made", "Missed")) +
  scale_x_continuous(limits = c(-27.5, 27.5)) + 
  scale_y_continuous(limits = c(0, 45)) +
  theme(plot.title = element_text(hjust = .5, size = 22, family = "Comic Sans MS", face = "bold", vjust = -4),
        plot.subtitle = element_text(hjust = .5, size = 10, family = "Comic Sans MS", face = "bold", vjust = -8),
        legend.position = c(.5, .85),
        legend.direction = "horizontal",
        legend.title = element_blank(),
        legend.text = element_text(hjust = .5, size = 10, family = "Comic Sans MS", face = "bold", colour = "white")) +
  ggtitle(label = "Buddy Boeheim vs. San Diego State",
          subtitle = "30 PTS | 4 REB | 7-10 3PT - 3/19/21")

ggdraw(p1) + theme(plot.background = element_rect(fill="gray20", color = NA)) 

ggsave("Buddy_SDSU.png", height = 6, width = 6, dpi = 300)
```




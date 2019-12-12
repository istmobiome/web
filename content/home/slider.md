+++
# Slider widget.
widget = "slider"  # See https://sourcethemes.com/academic/docs/page-builder/
headless = true  # This file represents a page section.
active = true  # Activate this widget? true/false
weight = 10  # Order that this section will appear.

# Slide interval.
# Use `false` to disable animation or enter a time in ms, e.g. `5000` (5s).
interval = "false"

# Slide height (optional).
# E.g. `500px` for 500 pixels or `calc(100vh - 70px)` for full screen.
height = "250px"

# Slides.
# Duplicate an `[[item]]` block to add more slides.
[[item]]
  title = "The Istmobiome Project"
  content = "a microbial tale told in two oceans"
  align = "center"  # Choose `center`, `left`, or `right`.

  # Overlay a color or image (optional).
  #   Deactivate an option by commenting out the line, prefixing it with `#`.
  # Add opacity value to change between light/dark mode
  # Otherwise 'FF' seems to keep color consistant
  overlay_color = "#1f4e74FF"  # An HTML color value. Extra digits to cotrol opacity
  overlay_img = ""  # Image path relative to your `static/img/` folder.
  overlay_filter = 0  # Darken the image. Value in range 0-1.

  # Call to action button (optional).
  #   Activate the button by specifying a URL and button label below.
  #   Deactivate by commenting out parameters, prefixing lines with `#`.
  # cta_label = "Get Academic"
  # cta_url = "https://sourcethemes.com/academic/"
  # cta_icon_pack = "fas"
  # cta_icon = "graduation-cap"

  [[item]]
    title = "The Isthmus of Panama Changed the World..."
    content = "Three million years ago an experiment began"
    align = "center"  # Choose `center`, `left`, or `right`.

    # Overlay a color or image (optional).
    #   Deactivate an option by commenting out the line, prefixing it with `#`.
    overlay_color = "#1f4e74FF"  # An HTML color value. Extra digits to cotrol opacity
    overlay_img = ""  # Image path relative to your `static/img/` folder.
    overlay_filter = 0  # Darken the image. Value in range 0-1.

[[item]]
  title = "...on Land..."
  content = "it connected two continents"
  align = "right"

  overlay_color = "#666"  # An HTML color value.
  overlay_img = "headers/land.jpg"  # Image path relative to your `static/img/` folder.
  overlay_filter = 0.3  # Darken the image. Value in range 0-1.

[[item]]
  title = "...and in the Sea."
  content = "it divided one ocean. "
  align = "right"

  overlay_color = "#333"  # An HTML color value.
  overlay_img = "headers/sea.jpg"  # Image path relative to your `static/img/` folder.
  overlay_filter = 0.3  # Darken the image. Value in range 0-1.
+++

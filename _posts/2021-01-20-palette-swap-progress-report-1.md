---
---

![Basic StateMachine](/assets/images/palette_swap.gif)  

Got some basic palette swapping to work. What you're seeing here is not an image swap, I'm actually using a fragment shader to swap 
the default green to either blue or red. 

Credits to [https://github.com/thomasgoldstein/zabuyaki](https://github.com/thomasgoldstein/zabuyaki){:target="_blank"} (very fun open source game FYI) for this shader. It took me a while to find it so hopefully when someone else googles "Love2D Palette Swap" they can find this post!
{% raw %}
~~~
local lume = require 'lume'
local function makePaletteShader(originalColors, alternateColors)
  assert(#originalColors == lume.count(alternateColors), 'OriginalColors and AlternateColors array length need to match for palette shader')
  if #originalColors == 0 then return nil end
  local count = #originalColors
  local shaderCode = 'const int colorCount = ' .. tostring(count) .. ';'  -- ironic that this is const lol
  shaderCode = shaderCode .. [[
    uniform vec4 originalColors[colorCount];
    uniform vec4 alternateColors[colorCount];
    
    vec4 effect(vec4 color, Image texture, vec2 texture_coords, vec2 screen_coords) {
      // This is the current pixel color
      vec4 pixel = Texel(texture, texture_coords); 
      for (int i = 0; i < colorCount; ++i) {
        if (pixel == originalColors[i]) {
          // return alternate color if it matches an specified original color
          return alternateColors[i] * color;
        }
      }
      // return default color if it does not match any of the designated original colors
      return pixel * color;
    }
  ]]
  local shader = love.graphics.newShader(shaderCode)
    if #self.originalColors == 1 then
      self.shader:sendColor('originalColors', self.originalColors[1])
      self.shader:sendColor('alternateColors', self.alternateColors[1])
    else
      local otherOrignalColors = lume.slice(self.originalColors, 1)
      local otherAlternateColors = lume.slice(self.alternateColors, 1)
      self.shader:sendColor('originalColors', self.originalColors[1], otherOrignalColors)
      self.shader:sendColor('alternateColors', self.alternateColors[1], otherAlternateColors)
    end
  end
~~~
{% endraw %}

In the future I plan on making palettes for tilesets as well. This will save time and memory since I won't have to create seperate images if I want a different
color palette for certain sprites. 


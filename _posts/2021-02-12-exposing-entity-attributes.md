---
---
If you're making a game, I think its critical that you implement some sort of entity inspector as soon as possible. Having it makes debugging so much easier since you can change entity member values during runtime instead:

* Make file change
* Saving
* Run game
* Repeat until you accomplished whatever it is you were trying to do    

This process is even worse if you're using a compiled language, which adds an extra step and wastes even more of your time.

Here is how I expose properties for my Entity classes in lua.
{% highlight lua %}
function Entity:getInspectorProperties()
  local props = InspectorProperties(self)
  -- expose a string variable that you dont want users to be editing
  props:addReadOnlyString('Name', 'name')
  -- expose a vector variable by providing a getter and setter function
  props:addVector2('Position', self.getPosition, self.setPosition)
  -- expose a vector2i variable
  props:addVector2i('Size', self.getSize, self.resize)
  return props
end
{% endhighlight %}

![Entity Inspector in game screenshot](/assets/images/entity_inspector_preview.png)  

Pretty simple stuff. Even if the language I'm using supports reflection and attributes this is still my 
go to method of exposing object values.
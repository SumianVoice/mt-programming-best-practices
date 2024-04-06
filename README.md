

# Mob code examples
## Direct control
This is the worst coding style. Duplication is everywhere, and you can't extend anything or make general alterations without having to change potentially hundreds of different implementations e.g. mobs.
```lua
mobdef = {
    on_step = function(self, dtime)
        local pos = self.object:get_pos()
        if self._target  then
            local tp = self._target:get_pos()
            if ((pos.x - tp.x)^2) + ((pos.z - tp.z)^2) + ((pos.y - tp.y)^2) > self._viewdistance^2 then
                self._target = nil
            end
        end
        for i, object in ipairs(minetest.get_objects_inside_radius(pos, self._viewdistance)) do
            local tp = object:get_pos()
            if ((pos.x - tp.x)^2) + ((pos.z - tp.z)^2) + ((pos.y - tp.y)^2) > self._viewdistance^2 then
                self._target = object
            end
        end
        if self._target then
            local tp = self._target:get_pos()
            if ((pos.x - tp.x)^2) + ((pos.z - tp.z)^2) + ((pos.y - tp.y)^2) > 2 then
                local dir = vector.direction(pos, tp)
                self.object:set_velocity(dir * self._speed)
            end
        end
    end,
    _speed = 5,
    _target = nil,
    _viewdistance = 30,
}
```

## Everything-is-an-api style
This is much better since the API controls the basic functionality but the mob still controls how to use that functionality.
```lua
-- mob_api will be a table with many functions in it, and is assumed to just "do what it says on the tin"
mobdef = {
    on_step = function(self, dtime)
        mob_api.get_target(self)
        if mob_api.has_target(self)
        and mob_api.get_distance_to_target2(self) > 2 then
            mob_api.move_to_target(self)
        end
    end,
    _speed = 5,
    _target = nil,
    _viewdistance = 30,
}
```

## Declaritive with APIs
Better SOMETIMES but not always.
Keep in mind this style / structure is NOT a replacement for being able to control things directly or use APIs. It is a way to not have to do extra coding and not have redundancy; this cannot exclude the flexibility of extension; it's a helper not a replacement.
```lua
-- mob_api will now also take more actions, assuming behavior based on what you've definedl; the mob_on_step function will branch out and do a lot of different functionality, and there would be the possibility of registering or overriding behaviors as well
mobdef = {
    _behaviors = {
        follow_players = {min_distance=2},
        attack_players = {damagegroups={}},
        run_on_low_health = {min_health=4},
    },
    on_step = function(self, dtime)
        mob_api.mob_on_step(self, dtime)
    end,
    _speed = 5,
    _target = nil,
    _viewdistance = 30,
}
```


# API examples
## Direct Control
Really bad, no extension
```lua
minetest.register_on_mods_loaded(function()
    for itemname, def in pairs(minetest.registered_items) do
        local t = minetest.colorize("#ffffaa", def.description)
        if def._desc_helptext then
            t = t .. "\n" .. def._desc_helptext
        end
        if def._desc_longdesc then
            t = t .. "\n" .. def._desc_longdesc
        end
        minetest.override_item(itemname, {description=t, _desc_old = def.description})
    end
end)
```

## Still bad, but more sneaky
This looks like it's better, but it's just as bad really. It's a more extensible system sure, but it still has major flaws since where it looks is hardcoded and so is how it adds it to the description. If you're going to add a system that overwrites some part of the game, you better make sure you can extend it or alter its behavior wherever possible or else it will be a fork and rewrite time bomb.
```lua
local desc_api = {}
local desc_tags = {}
function desc_api.register_desc_tag(tag, color)
    table.insert(desc_tags, {tag=tag, color=color})
end

minetest.register_on_mods_loaded(function()
    for itemname, def in pairs(minetest.registered_items) do
        local t = minetest.colorize("#ffffaa", def.description)
        for i, desc_tag in ipairs(desc_tags) do
            if def[desc_tag.tag] then
                if desc_tag.color then
                    t = t .. "\n" .. minetest.colorize(desc_tag.color, def[desc_tag.tag])
                else
                    t = t .. "\n" .. def[desc_tag.tag]
                end
            end
        end
        minetest.override_item(itemname, {description=t, _desc_old = def.description})
    end
end)
```

# Slightly better
Since we're using a callback, you can do whatever you like. You don't have to put your tags inside the node definition, you can add arbitrary bits that aren't unique to an item, and you can add details about the item (e.g. durability or the items internal name) that otherwise you would have had to hardcode into the tags if you used the previous system.
```lua
local desc_api = {}
local desc_processes = {}
function desc_api.register_desc_process(callback, priority)
    if priority then
        priority = math.min(math.max(priority, 1), #desc_processes + 1)
        table.insert(desc_processes, priority, callback)
    else
        table.insert(desc_processes, callback)
    end
end

minetest.register_on_mods_loaded(function()
    for itemname, def in pairs(minetest.registered_items) do
        local t = minetest.colorize("#ffffaa", def.description)
        for i, callback in ipairs(desc_processes) do
            local ret, color = callback(itemname, def)
            if color and ret then
                t = t .. "\n" .. minetest.colorize(callback.color, ret)
            elseif ret then
                t = t .. "\n" .. ret
            end
        end
        minetest.override_item(itemname, {description=t, _desc_old = def.description})
    end
end)
```


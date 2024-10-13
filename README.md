

# Mob code examples
## Direct control
This is the worst coding style. Duplication is everywhere, and you can't extend anything or make general alterations without having to change potentially hundreds of different implementations e.g. mobs.
```lua
local dist2 = function(p1, p2)
    return (p1.x - p2.x)^2 + (p1.z - p2.z)^2 + (p1.y - p2.y)^2
end
mobdef = {
    on_step = function(self, dtime)
        local pos = self.object:get_pos()
        if self._target  then
            local tp = self._target:get_pos()
            if dist2(pos, tp) > self._viewdistance^2 then
                self._target = nil
            end
        end
        for i, object in ipairs(minetest.get_objects_inside_radius(pos, self._viewdistance)) do
            local tp = object:get_pos()
            if dist2(pos, tp) > self._viewdistance^2 then
                self._target = object
            end
        end
        if self._target then
            local tp = self._target:get_pos()
            if dist2(pos, tp) > 2 then
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
Note: `mob_api` will be a table with many functions in it, and is assumed to just "do what it says on the tin"
```lua
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
Note: `mob_api` will now also take more actions, assuming behavior based on what you've defined; the `mob_on_step` function in the API will branch out and do a lot of different functionality, and there would be the possibility of registering or overriding behaviors as well.
```lua
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

## Slightly better
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


# System Design
This is all my preference and opinion, but I will say that it works. It works because Minetest is a procedurally programmed environment, it doesn't force you into some paradigm like object oriented or functional etc.

When designing a feature, think about what scope will have. What does it do? Narrow it down to one purpose, like "provide the ability to arbitrarily control how weapons deal damage". If you have to say "and" while describing what something does, split it up into two seperate things.

Then, think about how you will structure it. If you are thinking of using OOP (object oriented programming) or any other "paradigm", **don't**. If you know exactly *why* it will make it *better* and *how*, then go ahead and do it, otherwise never ever *ever* use some design structure just because you were told it's good or the best way or the only way. Programming is about making systems to solve problems, so don't add more problems to solve, you will have enough of those already. Specifically, for OOP, never use this unless your feature needs to have many things be created and destroyed over and over **and** they need to be independent and have their own seperate logic. If you don't know if that's the case or don't know what that means, don't use it.

### Example: "provide the ability to arbitrarily control how weapons deal damage"
In this case, the weapon will have a **definition**, and that could be used to store information about how the weapon deals damage. That's limited though, since you have to for example have your API check for `item._fire_damage` and then `item._ice_damage` and 400 other random effects or damage types. That doesn't scale well. Instead, use callbacks. A callback is a function that gets called when something happens. For example, the `on_place` callback in nodes, or the `on_step` callback in entities. These get called at certain times. For this weapon API example, we might have a `on_attack` and an `on_blocked` and an `on_step`. These would call when you think they would. `on_attack` is called when you hit someone with the weapon, and this allows you to deal damage or add status effects or whatever else you want to do. You don't have to rely on already implemented behaviours.

Every time you have to search in your code and update some old feature to accomodate new use cases is a 80% chance of spending another six hours debugging, so we're trying to make it so robust from the beginning that we can just override the behaviour from anywhere if we like.

Next is to design the actual systems. The callbacks are decided, so now we move on to making the actual functions. Think about the biggest and smallest functions. A function like `use_weapon(attacker, itemstack, pointed_thing)` is huge; it will do lots of things and call many things. This is the *top of the code pyramid*. The top of the pyramid cannot exist without the base of the pyramid, and so we have also the small functions, things like `get_target_from_raycast(pos1, pos2)`. These do not reference any other functions generally, they just do what they say they do. Generally these don't take actions or change the world (e.g. deal damage) unless it's super clear that it should. For example `deal_damage` should only deal damage. It should not apply status effects or do knockback or test for if the opponent is blocking. If it did, you'd have to call it `deal_damage_and_knockback_and_status_effects_if_opponent_not_blocking` and you can see why that would be a problem.

When we have the names and the parameters of the large functions which will be the "in" to the API, we build the supporting structure. You can think about the top of the pyramid functions as containing pseudocode-like calls.

```lua
function use_weapon(attacker, itemstack, pointed_thing)
    if not can_use_weapon(attacker, itemstack, pointed_thing) then return end
    if not is_creative(attacker) then
        wear_weapon(attacker, itemstack, pointed_thing)
    end
    local attack = get_attack_params(attacker, itemstack, pointed_thing)
    itemstack = send_attack(attacker, itemstack, attack, pointed_thing) or itemstack
    return itemstack
end
```

This reads like a sentence, and that is intentional. If we put the content of all these functions straight into the `use_weapon` function, a few things would happen:
1. it makes it unreadable, and it's hard to find stuff
2. the function no longer does only what it says it does, it does a heap of other random junk too
3. it is harder to make changes
4. you can't override one part of the chain of things being done, so if you need to stop item wear, you can't for example
5. it just looks bad
6. other mods or other parts of your game / mod can't access e.g. `send_attack` because it's hidden within a massive bloated function and isn't accessible

You can invert these to see what you should do, but I think it's better to have cautionary tales than "you should do this trust me bro". There is a reason "single responsibility" is a good principle, it's just misunderstood by most people. You can't define a "single thing", but you can tell there's a problem when your function name becomes `use_weapon_and_add_wear_if_creative_and_send_attack_by_item_definition`.

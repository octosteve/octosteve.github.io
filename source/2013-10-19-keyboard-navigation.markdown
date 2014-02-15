---
title: "Keyboard Navigation"
date: 2013-10-19 10:59
tags: [navigation, sharp axe]

---

## Get over here!

One of the unsung heroes of development is knowing how to use your keyboard. I don't just mean typing. That skill, while important, is still second to being able to move around. What do I mean by this?

Let's say I'm typing this obscenely long line of bash:

``` bash
cat ~/schol/GA/BEWD_Curriculum/01_Dev_Workflow/README.md| grep -i git
```

Oh NO! I've misspelled `school`. Here we go: backspace backspace backspace… 59 backspaces! Even if you didn't erase it and just used left, it's still going to take a while.

That's crazy. Let's try another way.

A keyboard shortcut you can use is holding the `option` or `alt` (written `⌥`) key and pressing `left`. Do that 11 times, and you'll get to the `G` in `GA`. From there, it's just 2 taps of the `left` key and you're good to go. 

There's another way… There's always another way…

You can jump to the beginning of the line by holding `Control` (written `Ctrl`, `^` ), and pressing `a`. From there, along with `option + right` you've brought it down to 5 taps (^a, ⌥+right, `right`, `right`, `right` ).

[This article](http://lifehacker.com/5743814/become-a-command-line-ninja-with-these-time+saving-shortcuts) has loads of cool stuff you can use to get around. 

My point is, make time to practice getting good. Think of it like golf. The less strokes, the better. Over time you'll see the lag between your intention and your computer screen shrink. Decide that it's important and it will make your development career amazing.

### There's always another way
This would work too 
`^schol^school`

You would submit the messed up line, it would give you an error. Then you would type that.

### Editing Text

A lot of these things work in your editor of choice. A lot of developers use [Sublime Text](http://www.sublimetext.com/2). 

Let's look at getting around a small ruby file. It's from my [slide_hero gem](https://github.com/StevenNunez/slide_hero). Don't worry too much about what it does.

``` ruby
module SlideHero                                                                   
  class Point                                                                      
    attr_reader :text                                                              
    SUPPORTED_ANIMATIONS = %w{grow shrink roll-in fade-out                         
      highlight-red highlight-green highlight-blue}                                
                                                                                   
    def initialize(text, animation: nil)                                           
      @text = text                                                                 
      @animation = animation                                                       
    end                                                                            
                                                                                   
    def compile                                                                    
      "<p#{animation}>#{text}</p>"                                                 
    end                                                                            
                                                                                   
    private                                                                        
    def animation                                                                  
      if @animation                                                                
        animation_markup = ' class="fragment '                                     
        if SUPPORTED_ANIMATIONS.include? @animation                                
          animation_markup << @animation                                           
        end                                                                        
        animation_markup + "\""                                                    
      end                                                                          
    end                                                                            
  end                                                                              
end                        
```

Let's say I want to jump to add another method (`def animation` is me defining a method). Since the editor usually opens at the top of the file, I'll have to press down 16 times or worse, USE MY MOUSE!  Looking at Sublime Text's `Goto` menu, I see ^G will let me specify what line to go to. Cool! Ok, well now I'm on the line with `private` and need to get *after* it to enter my new method. We can use one of our old buddies `⌥+right` here! OK, `⌥+right`, `enter`, start typing!

If you're planning on using Sublime Text, be sure to learn the [keyboard shortcuts!](http://docs.sublimetext.info/en/latest/reference/keyboard_shortcuts_osx.html). They're there because of you! Use them and be awesome.

Another time saving shortcut is ⌘D on a Mac. Here's how it works. Say I want to replace every time the word `text` appears in my code example. I could use our fancy navigation to line 3, then 8 then 13, BUT there's another way. If you put your cursor on `text` (any of them), and press `⌘D`, it will start to highlight every instance of `text`. Now just start typing. It will erase all of them. What a time saver.

In the end, it's your ability to move around effectively that will make you a great developer. All the other stuff you can learn as you go. There are times I have no idea how to solve something, but being able to run really fast experiments makes my life so much easier. 

That's all I have.

Happy Clacking. 




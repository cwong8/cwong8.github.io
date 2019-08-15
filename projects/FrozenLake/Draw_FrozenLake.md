---
layout: page
title: Draw FrozenLake
permalink: /projects/FrozenLake/Draw_FrozenLake
---


Arrow images from: [https://www.flaticon.com/packs/arrows-41](https://www.flaticon.com/packs/arrows-41)



### Main drawing and saving function


```python
# Packages
import numpy as np
from tkinter import *
from PIL import Image, ImageTk
import math
import itertools
# For saving images
from PIL import ImageDraw

def draw_canvas(qtable, height, width, save = False, filename = 'FrozenLake.png'):
    """
    Visualizes the FrozenLake environment with the option to save the images.
    
    Args:
        qtable: Q-table from reinforcement learning. Should have either 16 or 64 states for a 4x4 or 8x8 grid respectively.
        height: Height of output image.
        width: Width of output image.
        save: Set save = True to save output to current directory 
        filename: Filename of image to be saved
        
    Returns:
        Image of the FrozenLake environment with arrows pointing to directions chosen by the Q-table based on maximum value.
    """
    top = Tk()
    grid_dim = math.sqrt(qtable.shape[0])
    if int(grid_dim + 0.5) ** 2 != qtable.shape[0]:
        print("WARNING: State size is not a perfect square.")
        return
    else:
        grid_dim = int(grid_dim)
     
    # Arrow images scaled to canvas size
    left = ImageTk.PhotoImage(Image.open("./arrows/left-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2))))
    down = ImageTk.PhotoImage(Image.open("./arrows/down-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2))))
    right = ImageTk.PhotoImage(Image.open("./arrows/right-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2))))
    up = ImageTk.PhotoImage(Image.open("./arrows/up-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2))))
    arrows = [left, down, right, up] 
    
    # Max Q-value per row
    max_q = qtable.argmax(axis = 1)
    max_q = max_q.reshape(grid_dim, grid_dim)
    
    # tkinter canvas
    C = Canvas(top, bd = 0, bg = "white", height = height, width = width)
    
    # Draw grid
    for i, j in zip(range(grid_dim), range(grid_dim)):
        # Vertical lines
        C.create_line(j*(width/grid_dim), 0, j*(width/grid_dim), height)
        # Horizontal lines
        C.create_line(0, i*(height/grid_dim), width, i*(height/grid_dim))
    
    # Color holes red
    if grid_dim == 4:
        holes = [(1, 1), (1, 3), (2, 3), (3, 0)]
    elif grid_dim == 8:
        holes = [(2, 3), (3, 5), (4, 3), (5, 1), (5, 2), (5, 6), (6, 1), (6, 4), (6, 6), (7, 3)]
    else:
        print("What frozen lake are you using?")
        return
    for hole in holes:
        C.create_polygon(hole[1]*(width/grid_dim), hole[0]*(height/grid_dim), hole[1]*(width/grid_dim), (hole[0]+1)*(height/grid_dim), (hole[1]+1)*(width/grid_dim), (hole[0]+1)*(height/grid_dim), (hole[1]+1)*(width/grid_dim), hole[0]*(height/grid_dim), fill = "red")
   
    # Center our arrow images in the grid and draw them
    # Our grid indices looks like this for a 4x4 grid
    #  0 |  1 |  2 |  3
    # -----------------
    #  4 |  5 |  6 |  7
    # -----------------
    #  8 |  9 | 10 | 11
    # -----------------
    # 12 | 13 | 14 | 15
    # -----------------   
    for i, j in itertools.product(range(grid_dim), range(grid_dim)):
        C.create_image(j*(width/grid_dim) + (height/(grid_dim*2)), i*(height/grid_dim) + (width/(grid_dim*2)), image = arrows[max_q[i, j]])
    
    if save == True:
        white = (255, 255, 255)
        black = (0, 0, 0)
        image = Image.new("RGB", (width, height), white)
        draw = ImageDraw.Draw(image)
        
        # Draw grid
        for i, j in zip(range(grid_dim), range(grid_dim)):
            # Vertical lines
            draw.line([j*(width/grid_dim), 0, j*(width/grid_dim), height], black)
            # Horizontal lines
            draw.line([0, i*(height/grid_dim), width, i*(height/grid_dim)], black)
        
        # Color holes red
        for hole in holes:
            draw.polygon([hole[1]*(width/grid_dim), hole[0]*(height/grid_dim), hole[1]*(width/grid_dim), (hole[0]+1)*(height/grid_dim), (hole[1]+1)*(width/grid_dim), (hole[0]+1)*(height/grid_dim), (hole[1]+1)*(width/grid_dim), hole[0]*(height/grid_dim)], fill = "red")
            
       # Center our arrow images in the grid and draw them
        # Our grid indices looks like this for a 4x4 grid
        #  0 |  1 |  2 |  3
        # -----------------
        #  4 |  5 |  6 |  7
        # -----------------
        #  8 |  9 | 10 | 11
        # -----------------
        # 12 | 13 | 14 | 15
        # -----------------   
        for i, j in itertools.product(range(grid_dim), range(grid_dim)):
            image.paste(im = arrows[max_q[i, j]], box = [j*(width//grid_dim) + (height//(grid_dim*4)), i*(height//grid_dim) + (width//(grid_dim*4))])
        
        # Save image
        image.save(filename)
    
    C.pack()
    top.mainloop()
```

### Separate saving function if that's your style


```python
from PIL import ImageDraw
# Saves images through PIL by mimicing canvas drawings
def save_canvas(qtable, height, width, filename = 'FrozenLake.png'):
    root = Tk()
    grid_dim = math.sqrt(qtable.shape[0])
    if int(grid_dim + 0.5) ** 2 != qtable.shape[0]:
        print("WARNING: State size is not a perfect square.")
        return
    else:
        grid_dim = int(grid_dim)
    
    white = (255, 255, 255)
    black = (0, 0, 0)
    image = Image.new("RGB", (width, height), white)
    draw = ImageDraw.Draw(image)
    
    # Arrow images scaled to canvas size
    left = Image.open("./arrows/left-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2)))
    down = Image.open("./arrows/down-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2)))
    right = Image.open("./arrows/right-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2)))
    up = Image.open("./arrows/up-arrow.png").resize((width//(grid_dim*2), height//(grid_dim*2)))
    arrows = [left, down, right, up] 
    
    # Max Q-value per row
    max_q = qtable.argmax(axis = 1)
    max_q = max_q.reshape(grid_dim, grid_dim)
    
    # Draw grid
    for i, j in zip(range(grid_dim), range(grid_dim)):
        # Vertical lines
        draw.line([j*(width/grid_dim), 0, j*(width/grid_dim), height], black)
        # Horizontal lines
        draw.line([0, i*(height/grid_dim), width, i*(height/grid_dim)], black)
        
    # Color holes red
    if grid_dim == 4:
        holes = [(1, 1), (1, 3), (2, 3), (3, 0)]
    elif grid_dim == 8:
        holes = [(2, 3), (3, 5), (4, 3), (5, 1), (5, 2), (5, 6), (6, 1), (6, 4), (6, 6), (7, 3)]
    else:
        print("What frozen lake are you using?")
        return
    for hole in holes:
        draw.polygon([hole[1]*(width/grid_dim), hole[0]*(height/grid_dim), hole[1]*(width/grid_dim), (hole[0]+1)*(height/grid_dim), (hole[1]+1)*(width/grid_dim), (hole[0]+1)*(height/grid_dim), (hole[1]+1)*(width/grid_dim), hole[0]*(height/grid_dim)], fill = "red")

   # Center our arrow images in the grid and draw them
    # Our grid indices looks like this for a 4x4 grid
    #  0 |  1 |  2 |  3
    # -----------------
    #  4 |  5 |  6 |  7
    # -----------------
    #  8 |  9 | 10 | 11
    # -----------------
    # 12 | 13 | 14 | 15
    # -----------------   
    for i, j in itertools.product(range(grid_dim), range(grid_dim)):
        image.paste(im = arrows[max_q[i, j]], box = [j*(width//grid_dim) + (height//(grid_dim*4)), i*(height//grid_dim) + (width//(grid_dim*4))])
    
    # Save image
    image.save(filename)
        
        
```

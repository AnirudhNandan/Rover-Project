# Project: Search and Sample Return

## Notebook Analysis:

### 1. Creating function for colour selection of obstacles and rock samples:

#### Colour selection of obstacles:

This function sets colour channels to 1 if they are above the chosen threshold or to 0 if they are otherwise.

Code:
```python
def color_thresh(img, rgb_thresh=(0,0,0)):
    color_select = np.zeros_like(img[:,:,0])
    color_select[img[:,:,0]>rgb_thresh[0]] = 1
    color_select[img[:,:,1]>rgb_thresh[1]] = 1
    color_select[img[:,:,2]>rgb_thresh[2]] = 1
    return color_select
    
# Define color selection criteria
###### TODO: MODIFY THESE VARIABLES TO MAKE YOUR COLOR SELECTION
red_threshold = 160
green_threshold = 160
blue_threshold = 160
######
rgb_threshold = (red_threshold, green_threshold, blue_threshold)

# pixels below the thresholds
colorsel = color_thresh(image, rgb_thresh=rgb_threshold)
```

#### Rock colour selection:

Rock is visually located through thresholding by choosing suitable colour threshold value.

Code:
```python 
def find_rocks(img, levels=(110,110,50)):
    rockpix = ((img[:,:,0] > levels[0]) \
                & (img[:,:,1] > levels[1]) \
                & (img[:,:,2] < levels[2]))
    
    color_select = np.zeros_like(img[:,:,0])
    color_select[rockpix] = 1
    
    return color_select
    
rock_map = find_rocks(warped, levels=(110, 110, 50))
```
### 2. Creating process_image( ) function:

This function converts raw images to perspective transformed images and identifies navigable terrain, obstacles, and rock samples. This function also maps navigable terrain, obstacles, and rock samples in world map. Output video included in repo.

Code:
```python
def process_image(img):
    # Perspective Transform and mask
    warped, mask = perspect_transform(img, source, destination)
    # Apply color threshold to identify navigable terrain/obstacles/rock samples
    threshed = color_thresh(warped)
    # Create obstacle map
    obs_map = np.absolute(np.float32(threshed) - 1) * mask
    # Create rover based co-ordinate map
    xpix, ypix = rover_coords(threshed)
    
    world_size = data.worldmap.shape[0]
    scale = 2 * dst_size
    xpos = data.xpos[data.count]
    ypos = data.ypos[data.count]
    yaw = data.yaw[data.count]
    # Convert to world co-ordinates
    x_world, y_world = pix_to_world(xpix, ypix, xpos, ypos,
                                   yaw, world_size, scale)
    obsxpix, obsypix = rover_coords(obs_map)
    # Convert obstacle map to world co-ordinates
    obs_x_world, obs_y_world = pix_to_world(obsxpix, obsypix, xpos, ypos,
                                           yaw, world_size, scale)
    
    data.worldmap[y_world, x_world, 2] = 255
    data.worldmap[obs_y_world, obs_x_world, 0] = 255
    nav_pix = data.worldmap[:,:,2] > 0
    data.worldmap[nav_pix, 0] = 0
    # See if we can find some rocks
    rock_map = find_rocks(warped, levels=(110, 110, 50))
    if rock_map.any():
        rock_x, rock_y = rover_coords(rock_map)
        rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, xpos,
                                                  ypos, yaw, world_size, scale)
        data.worldmap[rock_y_world, rock_x_world, :] = 255
        
    # 7) Make a mosaic image, below is some example code
        # First create a blank image (can be whatever shape you like)
    output_image = np.zeros((img.shape[0] + data.worldmap.shape[0], img.shape[1]*2, 3))
        # Next you can populate regions of the image with various output
        # Here I'm putting the original image in the upper left hand corner
    output_image[0:img.shape[0], 0:img.shape[1]] = img

        # Let's create more images to add to the mosaic, first a warped image
    #warped = perspect_transform(img, source, destination)
        # Add the warped image in the upper right hand corner
    output_image[0:img.shape[0], img.shape[1]:] = warped

        # Overlay worldmap with ground truth map
    map_add = cv2.addWeighted(data.worldmap, 1, data.ground_truth, 0.5, 0)
        # Flip map overlay so y-axis points upward and add to output_image 
    output_image[img.shape[0]:, 0:data.worldmap.shape[1]] = np.flipud(map_add)


        # Then putting some text over the image
    cv2.putText(output_image,"Populate this image with your analyses to make a video!", (20, 20), 
                cv2.FONT_HERSHEY_COMPLEX, 0.4, (255, 255, 255), 1)
    if data.count < len(data.images) - 1:
        data.count += 1 # Keep track of the index in the Databucket()
    
    return output_image
```

Results:

Perspective transform:
![1](https://user-images.githubusercontent.com/7349926/34482669-7b0b34da-ef87-11e7-803e-b1bae65aef0c.png)

![2](https://user-images.githubusercontent.com/7349926/34482693-ab8f74a4-ef87-11e7-9cb8-299772550db3.png)
Colour thresholding:
![3](https://user-images.githubusercontent.com/7349926/34482702-b65ca1ae-ef87-11e7-9183-009459cb8f47.png)
Navigation angle calculation:
![4](https://user-images.githubusercontent.com/7349926/34482713-bf916b2e-ef87-11e7-8fca-134be59bd410.png)
### Autonomous Navigation and Mapping:

#### perception_step( ) function:
The function perception_step( ) is filled in to work on Rover object of class RoverState( ). This function first perform perspective transform, then performs colour thresholding, updates Rover.vision_image, converts map image pixel values to rover-centric coordinates, converts rover centric pixel values to world coordinates, updates Rover worldmap, converts rover centric pixel positions to polar coordinates, updates Rover navigation angles, and plots rock positions.

Code:
```python
def perception_step(Rover):
    # Defining source and destination points for perspective transform
    dst_size = 5
    bottom_offset = 6
    image = Rover.img
    source = np.float32([[14, 140], [301, 140], [200, 96], [118, 96]])
    destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
                  [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
                  [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset], 
                  [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
                  ])
    # Applying perspective transform
    warped, mask = perspect_transform(Rover.img, source, destination)
    # Applying color threshold to identify navigable terrain/obstacles/rock samples
    threshed = color_thresh(warped)
    obs_map = np.absolute(np.float32(threshed) - 1)*mask
    # Updating Rover.vision_image
    Rover.vision_image[:,:,2] = threshed * 255
    Rover.vision_image[:,:,0] = obs_map * 255
    # Converting map image pixel values to rover-centric coordinates
    xpix, ypix = rover_coords(threshed)
    world_size = Rover.worldmap.shape[0]
    scale = 2 * dst_size
    # Converting rover-centric pixel values to world coordinates
    x_world, y_world = pix_to_world(xpix, ypix, Rover.pos[0], Rover.pos[1],
                                   Rover.yaw, world_size, scale)
    obsxpix, obsypix = rover_coords(obs_map)
    obs_x_world, obs_y_world = pix_to_world(obsypix, obsypix, Rover.pos[0], Rover.pos[1],
                                    Rover.yaw, world_size, scale)
    # Updating Rover worldmap
    Rover.worldmap[y_world, x_world, 2] += 10
    Rover.worldmap[obs_y_world, obs_x_world, 0] += 1
    # Converting rover-centric pixel positions to polar coordinates
    dist, angles = to_polar_coords(xpix, ypix)
    # Updating rover angles
    Rover.nav_angles = angles
    # Identifying and plotting rock positions
    rock_map = find_rocks(warped, levels=(110, 110, 50))
    if rock_map.any():
        rock_x, rock_y = rover_coords(rock_map)
        rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, Rover.pos[0],
                                                  Rover.pos[1], Rover.yaw, world_size, scale)
        rock_dist, rock_ang = to_polar_coords(rock_x, rock_y)
        rock_idx = np.argmin(rock_dist)
        rock_xcen = rock_x_world[rock_idx]
        rock_ycen = rock_y_world[rock_idx]

        Rover.worldmap[rock_ycen, rock_xcen, 1] = 255
        Rover.vision_image[:,:,1] = rock_map * 255
        # Modifications:
        Rover.rock_flag = True
    else:
        Rover.vision_image[:,:,1] = 0

    return Rover
```
#### decision_step( ) function:
The function decision_step( ) is used to manipulate data encapsulated in Rover object. This function acts the output of perception_step( ) to set throttle, brake, and steer values. The comments are provided in attached decision.py file.

### Launching Autonomous Mode:

#### Settings:

OS: Mac

Screen resolution: 800x600

Graphics quality: Good

Windowed: Yes

FPS: 23 to 34

#### Result:
Mapped: 75%

Found rocks: 3

Fidelity: 55%

# Stereo Music Player
**ECE 4180 Fall 2023 Final Project**<br>
**Team Members:** Ryan Su, Zili He

![project setup image](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/image/speaker.png)

## Table of Contents
- [Overview](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/README.md#overview)
- [Parts List](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/README.md#parts-list)
- [Hardware Setup](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/README.md#hardware-setup)
- [Software Setup](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/README.md#software-setup)
- [Insturction of Use](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/README.md#instruction-of-use)
- [Demo Video](https://github.com/Tianrui-RyanSu/ECE4180_F23_StereoMusicPlayer/blob/main/README.md#demo-video)

## Overview
This project implements a bluetooth controlled stereo music player. The music player loads audio files from an micro-sd card,
which stores both the left and right channels of the songs. The audio files are 8 bit at 8000KHz sample rate ".wav" files. 
The music player is controlled by BluefruitConnect app which establishes a bluetooth connection with the system. The music player
can play/pause and skip forward/backward songs, and can shuffle/order play the entire playlist on the micro-sd card. An LCD display
shows essential playback information like play status (play/pause), play modes (shuffle/order), song name, and song progess bar.
There is also a potentiometer connected to each of the speakers to enable volume control. Two 5V power supplies are needed to power
the 2 speakers, and an additional USB connection is needed from mbed to a computer for power and code changes. 

## Parts List

## Hardware Setup

## Software Setup
```cpp

#include "mbed.h"
#include "rtos.h"
#include "SDFileSystem.h"
#include "wave_player.h"
#include "wave_player_pwm.h"
#include "uLCD_4DGL.h"
#include <string>
#include <algorithm>

#define NUMSONGS 6     // number of songs on SD card

SDFileSystem sd1(p5, p6, p7, p8, "SD"); 
uLCD_4DGL uLCD(p9, p10, p30);
Serial blue(p28, p27);
DigitalOut led1(LED1);
AnalogOut DACout(p18);
PwmOut PWMout(p21);
wave_player waver_left(&DACout);
wave_player_pwm waver_right(&PWMout);

Timer t; 
volatile int songId = 0;
volatile int songQueue[NUMSONGS] = {0, 1, 2, 3, 4, 5};
float song_len[NUMSONGS] = {200, 150, 200, 200, 200};
string song_title_line1[NUMSONGS] = {"1.shape of", "2.half the", "3.let it", "4.never say", 
                               "5.no surprises", "6.perfect"};
string song_title_line2[NUMSONGS] = {"my heart", "world away", "be ", "never", " ", " "};

string song_path[NUMSONGS] = {"sting", "aurora", "beatles", "fray", "radiohead", "sheeran"};

FILE *wave_file_left, *wave_file_right;

Mutex serial_mtx, timer_mtx, file_mtx, play_mtx, print_mtx;

volatile bool play = false;
volatile bool skip = false;
volatile int error_left = 0;
volatile int error_right = 0;
volatile int finished_left = 1; 
volatile int finished_right = 1;
volatile bool shuffle = false;

char msg[256];

// Display progress bar and status
void LCD_ctrl(void const *argument)
{   
    int ind;  

    while(1)
    {
        timer_mtx.lock();
        ind = (((float) t.read() / song_len[songId]) * 112.0 * 0.83) + 8;
        timer_mtx.unlock();
        
        serial_mtx.lock();
        uLCD.circle(34, 25, 20, WHITE);
        uLCD.circle(94, 25, 20, WHITE);

        if (shuffle) {
            uLCD.filled_circle(94, 25, 15, 0xFFFF8F);
        } else {
            uLCD.filled_circle(94, 25, 15, 0x0ABAB5);
        }

        if (play) {
            uLCD.filled_circle(34, 25, 18, BLACK);
            uLCD.triangle(26, 15, 26, 35, 46, 25, WHITE);
        }            
        else {    
            uLCD.filled_circle(34, 25, 18, BLACK);            
            uLCD.rectangle(26, 15, 30, 35, WHITE);
            uLCD.rectangle(38, 15, 42, 35, WHITE);
        } 
        
        uLCD.locate(4, 13);
        timer_mtx.lock();
        uLCD.printf("%d:%02d/%d:%02d", (int) t.read()/60, (int) t.read() % 60,
                    (int) (song_len[songId]/0.83) / 60, (int) (song_len[songId]/0.83) % 60);
        timer_mtx.unlock();
        
        uLCD.rectangle(8, 116, 120, 120, WHITE);
        uLCD.filled_rectangle(9, 117, ind, 119, WHITE);
            
        serial_mtx.unlock();
        
        // Run every 0.25 sec
        Thread::wait(87);
    }
}

// Play the music for the current song ID 
void play_left(void const *argument)
{
    string path_left;
    
    while(1)
    {          
        finished_left = 1;
        error_left = 1;
        
        path_left = "/SD/";
        path_left += song_path[songQueue[songId]];
        path_left += "_left.wav";
        
        file_mtx.lock();
        waver_left.no_end_song();
        wave_file_left = fopen(path_left.c_str(),"r");
        if (wave_file_left == NULL) printf("file open error!\n\n\r");
        else printf("successfully opened file in left\n\n\r");
        file_mtx.unlock();

        serial_mtx.lock();
        uLCD.filled_rectangle(5, 40, 127, 100, BLACK);
        serial_mtx.unlock();

        int len = song_title_line1[songQueue[songId]].length();
        int pixels = 128 - (len * 8);
        
        if (shuffle) {
            serial_mtx.lock();
            uLCD.locate(pixels/14 + 1, 7);
            uLCD.printf("Shuffled");
            serial_mtx.unlock();
        } else {
            serial_mtx.lock();
            uLCD.locate(pixels/14 + 1, 7);
            uLCD.printf("Ordered ");
            serial_mtx.unlock();
        }
        serial_mtx.lock();
        uLCD.locate(pixels/14, 10);
        uLCD.text_height(1);
        uLCD.text_width(1);
        uLCD.printf(song_title_line1[songQueue[songId]].c_str());
        uLCD.locate(pixels/14 + 1, 11);
        uLCD.printf(song_title_line2[songQueue[songId]].c_str());
        serial_mtx.unlock();
        
        finished_left = waver_left.play(wave_file_left);

        if (!skip)
            songId = (songId < NUMSONGS - 1) ? (songId + 1) : 0; 
        else {
            waver_left.end_song();
            skip = false;
        }

        while(finished_left != 0 || finished_right != 0)
            printf("waiting on finishing\n\n\r");
        
        file_mtx.lock();
        error_left = fclose(wave_file_left);
        printf("left closed with %d.\n\n\r", error_left);
        file_mtx.unlock();

        while(error_left != 0 || error_right != 0) {
            printf("waiting on closing\n\n\r");
            if (error_left != 0) {
                file_mtx.lock();
                error_left = fclose(wave_file_left);
                file_mtx.unlock();
            }
        }
            
        
        serial_mtx.lock();
        uLCD.filled_rectangle(8, 116, 127, 120, BLACK);
        serial_mtx.unlock();
        
        timer_mtx.lock();
        t.reset();
        timer_mtx.unlock();

        Thread::wait(50);         
    }      
}

void play_right(void const *argument)
{
    string path_right;
    
    while(1)
    {          
        finished_right = 1;
        error_right = 1;

        path_right = "/SD/";
        path_right += song_path[songQueue[songId]];
        path_right += "_right.wav";
        
        file_mtx.lock();
        waver_right.no_end_song();
        wave_file_right = fopen(path_right.c_str(), "r");
        if (wave_file_right == NULL) printf("file open error!\n\n\r");
        else printf("successfully opened file in right\n\n\r");
        file_mtx.unlock();
            
        finished_right = waver_right.play(wave_file_right);

        if (!skip)
            // Increment songId to next song
            songId = (songId < NUMSONGS - 1) ? (songId + 1) : 0; 
        else {
            waver_right.end_song();
            skip = false;
        }

        while(finished_left != 0 || finished_right != 0)
            printf("waiting on finishing\n\n\r");
        
        file_mtx.lock();
        error_right = fclose(wave_file_right);
        printf("right closed with %d\n\n\r", error_right);
        file_mtx.unlock();

        while(error_left != 0 || error_right != 0) {
            printf("waiting on closing\n\n\r");
            if (error_right != 0) {
                file_mtx.lock();
                error_right = fclose(wave_file_right);
                file_mtx.unlock();
            }
        } 

        Thread::wait(50);         
    }    
}


int main() 
{    
    uLCD.cls();
    uLCD.baudrate(9600);

    PWMout.period(1.0/400000.0);
    
    waver_left.pause();
    waver_left.no_end_song();
    
    waver_right.pause();
    waver_right.no_end_song();
    
    Thread t1(LCD_ctrl);
    Thread t2(play_left);
    Thread t3(play_right);

    char bnum=0;
    char bhit=0;

    int blue_count = 0;

    while(1) {
        // If it's valid
        if(blue.readable()) {
            blue_count += 1;
            serial_mtx.lock();
            printf("current song: %d, %d. %d\n\n\r", songQueue[songId], songId, blue_count);
            if (blue.getc()=='!') {
                if (blue.getc()=='B') { //button data packet
                    bnum = blue.getc(); //button number
                    bhit = blue.getc(); //1=hit, 0=release
                    if (blue.getc()==char(~('!' + 'B' + bnum + bhit))) { //checksum OK?
                        switch (bnum) {
                            case '1': // play @ number_1
                                if (bhit=='1' && !play) {
                                    led1 = 1;

                                    waver_left.resume();  // Enable wave player
                                    waver_right.resume();

                                    timer_mtx.lock();
                                    t.start();              // Enable timer
                                    timer_mtx.unlock();
                                    
                                    play = true;
                                }
                                break;
                            case '2': // pause @ number_2
                                if (bhit=='1' && play) {
                                    led1 = 0;

                                    waver_left.pause();   // Disable wave player
                                    waver_right.pause();
                                    
                                    timer_mtx.lock();
                                    t.stop();               // Stop the timer
                                    timer_mtx.unlock();
                                    
                                    play = false;
                                }
                                break;
                            case '7': // back_skip @ left_arrow
                                if (bhit=='1' && play) {
                                    // Go back 1 song
                                    if (blue_count % 2 == 0)
                                        songId = (songId == 0) ? NUMSONGS - 1 : songId - 1;

                                    skip = true;
                                    
                                    waver_left.end_song();
                                    waver_right.end_song();
                                }
                                break;
                            case '8': // foward_skip @ right_arrow
                                if (bhit=='1' && play) {
                                    // Go forward 1 song
                                    if (blue_count % 2 == 0)
                                        songId = (songId >= NUMSONGS - 1) ? 0 : songId + 1;
                                    
                                    skip = true;

                                    waver_left.end_song();
                                    waver_right.end_song();
                                }
                                break;
                            case '3': // shuffled @ number_3
                                if (bhit=='1' && play) {
                                    random_shuffle(songQueue, songQueue + NUMSONGS);
                                    skip = true;
                                    shuffle = true;

                                    waver_left.end_song();
                                    waver_right.end_song();
                                }
                                break;
                            case '4': // ordered @ number_4
                                if (bhit=='1' && play) {
                                    sort(songQueue, songQueue + NUMSONGS);
                                    skip = true;
                                    shuffle = false;

                                    waver_left.end_song();
                                    waver_right.end_song();
                                }
                                break;
                                case '5': // arrow up
                                if (bhit=='1' && play) {
                                    NVIC_SystemReset();
                                }
                                break;
                            default:
                                break;
                        }
                    }
                }
            }
            serial_mtx.unlock();
        }     
        // Poll every 0.075 sec
        Thread::wait(125);
    }
}
```

## Instruction of Use
**Hardware (i.e. mbed and circuits)**
1. Reset button on mbed to reset the system and reload programs stored in mbed.
2. Twist the 2 potentiometers attached to each speaker for per-channel volume control.

**Software (i.e. BluefruitConnect App)**
1. Left/right button on the interface for skip backward/forward, respectively.
2. Numerical button 1 to play or resume after pause audio playback.
3. Numerical button 2 to pause audio playback.
4. Numerical button 3 to enable shuffled playlist and play in shuffled mode.
5. Numerical button 4 to disable shuffled mode to play music in original order.
6. Up button for a software synchronized system reset.

## Demo Video
[![Watch the video](https://img.youtube.com/vi/6G0gq5Kxjgo/0.jpg)](https://www.youtube.com/watch?v=6G0gq5Kxjgo)


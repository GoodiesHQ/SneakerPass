# SneakerPass

#### Password Management and more via Sneakernet

**SneakerPass** is my first open source hardware project. **SneakerPass** is a hand-held device that stores encrypted password information and is able to use native USB to act as a HID (keyboard) and send the password to the computer when requested by the user. Saving passwords in browsers opens your computer to a plethora of post-exploitation tools that identify and recover saved passwords. Simply plug in SneakerPass, enter your PIN code, and scroll through the "Passwords" menu to deploy your username/password combination.

## Construction - Total: < $50
I've outlined each component and my reasoning for using it.

#### Teensy 3.5/3.6 - $25
The logic of SneakerPass must be controlled by a microcontroller with USB HID capabilities and sufficient onboard storage for a large number of passwords (EEPROM is only 2kb on a Teensy 3.1 and 4kb on 3.5/3.6). Native USB is not required, thanks to libraries like [Project HID](https://github.com/NicoHood/HID). However, HID capability is necessary to act as a keyboard and send keystrokes to the host. *Please note that most Arduino devices do NOT have native USB. You will need to replace the bootloader.* Since the Teensy 3.5/3.6 offers native USB and onboard storage, and is extremely cost effective, I decided to go with that.

![Teensy 3.5](https://i.imgur.com/3qCAATn.png)

#### 4x4 Matrix Keypad - $8
In order to maintain authorized access to your passwords, you need to utilize a pin! This pin code is also used in the encryption/decryption process of storing passwords. Another important function of **SneakerPass** is the ability to navigate through the display menu. In the interest of security, I chose a 4x4 keypad (over a 4x3) because it contains four additional characters: A, B, C, D. This adds more characters for the pin code and allows a more intuitive navigation for the interface. Note that these could, however, be easily re-programmed if you desire a smaller form factor with a 4x3 keypad.

| Function  | 4x4 Keypad | 4x3 Keypad |
| --------- |:----------:| ----------:|
| UP        | A          | 2          |
| DOWN      | B          | 5          |
| MISC1     | C          | 3          |
| MISC2     | D          | 6          |
| SELECT    | #          | #          |
| BACK      | *          | *          |
| Picture   | ![4x4 Keypad](https://i.imgur.com/zZSavqJ.png) | ![4x3 Keypad](https://i.imgur.com/0lnXfcU.png) |


#### I2C OLED Display - $8

Teensy devices allow for I2C (inter-integrated circuits) connections. LCD displays can be used with normal GPIO pins, but I honestly felt that an OLED display provides more visual granularity and satisfaction, and was ultimately more intuive than a 2 or 4 line LCD screen. It has a bit of a retro look with a menu/navigation path in the top (yellow) portion and the remainder of the menu in the bottom (blue) portion. Because the Teensy 3.5/3.6 run at 3.3 volts, I chose a small form factor 0.96" SSD1306-compatible OLED display that operates in the 3.3-5V range. **Important:** The Teensy 3.6 is NOT 5V TOLERANT! Do not apply 5V to ANY pin on the Teensy 3.6. 

![OLED Display](https://i.imgur.com/Se4aocs.png)


#### Housing - $?
To be determined.

## Security/Encryption

Considering **SneakPass** is intended to be a portable password management system, security of the device is of utmost importance. From the file format to memory management, security is key.

#### Password Storage
For this project, I decided to create a very simple file format. The 10 byte header is constructed as follows:

| 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| 73 | 7c | 00 | 01 | 00 | 01 | 01 | 01 | 01 | 01 |

    0:  Magic byte. Should always have the value 0x73
    1:  Magic byte. Should always have the value 0x7c
    2:  Version (major). Indicates the major version release. Determines file compatibility.
    3:  Version (minor). Indicates the minor version release. Determines file compatibility.
    4:  AES Block Size. 80 = 128, C0 = 192, 00 = 256.
    5:  "Name" attribute block count. (+1 block for AES IV)
    6:  "URL" attribute block count. (+1 block for AES IV)
    7:  "Username" attribute block count. (+1 block for AES IV)
    8:  "Password" attribute block count. (+1 block for AES IV)
    9:  "Tertiary" attribute block count. (+1 block for AES IV)
    
Verifying the block count and file size from only the header should be trivial. Here is some pseudocode:

    unsigned char header[10];
    /* read 10 bytes from file to header */
    
    unsigned long total_bytes, block_size, block_count = 0;
    switch(header[4])
    {
      case '\x80':
        block_size = 0x80UL;
        break;
      case '\xc0':
        block_size = 0xc0UL;
        break;
      case '\x00':
        block_size = 0x100UL;
        break;
      default:
        perror("Invalid Block Size");
        exit(1);
    }
    
    for(int i = 5; i <= 9; ++i)
    {
      block_count += (unsigned long)header[i] + 1;
    }
    
    total_bytes = sizeof(header) + (block_size * block_count);
   
#### Hashing

I chose to use AES-CBC as the algorithm for storing passwords. Because the key needs to be an exact number of bytes, I have gone with the traditional method of using a cryptographic hash. However, seemingly no strong cryptographic hash supports 128, 192, and 256 bit block sizes. To circumvent this, I have decided to use the Keccak (SHA-3) 256 algorithm for all hashing. When using smaller key sizes, I truncate the hash to the first 128/192 bits respectively.

#### Pin Storage

Because the SHA3 hash is used as the encryption key, I decided to simply store the pin's double-hashed (`SHA3(SHA3(pin))`) value in EEPROM. In order to derive the encryption key, the pin code is hashed. In order to determine if it is valid, it is hashed again and compared to the key located in EEPROM storage.


#### Plaintext Password Management

Passwords are stored on the SD card in an encrypted format. However, in order for the Teensy to be able to type the correct keystrokes, the password must be held in plaintext for some amount of time. **SneakerPass** currently uses an item entry block size limit of `0x10` (16 blocks) meaning each value stored in the entry cannot exceed 255/383/511 bytes. There must be space for at least one additional byte to indicate padding. When reading **SneakFile**s to display to the user, ONLY the `name` portion of the entry is decrypted and displayed to the user. Once the name is selected, a buffer (equal to the aforementioned maximum file size) is created on the stack and the entire encryted file is moved in. When the user indicates they want the password to be sent to the host machine, it will be decrypted in-place. When the user indicates that they want to go back to the previous menu, the entire contents are overwritten with 0x00's before being deallocated from the stack. 

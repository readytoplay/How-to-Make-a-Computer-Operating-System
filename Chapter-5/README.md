## Chapter 5: Base classes for managing x86 architecture

Now that we know how to compile our C++ kernel and boot the binary using GRUB, we can start to do some nice stuffs in C/C++.

#### Printing to the screen console

We are going to use VGA default mode (03h) to display some text to the user. The screen can be directly access using the video memory at 0xB8000. The screen resolution is 80x25 and each caracters in the screen is defined by 2 bytes: one for the caracter and one for the style flag. So the size of the video memory is 4000 = 80*25*2

In the IO class ([io.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc)),:
* **x,y**: define the cursor position on the screen
* **real_screen**: define the  video memory pointer
* **putc(char c)**: print a unique caracter on the screen and manage cursor position
* **printf(char* s, ...)**: print a string

We add a method **putc** to the [IO Class](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc) to put a caracter on the screen and update the x,y position.

```cpp
/* put a byte on screen */
void Io::putc(char c){
	kattr = 0x07;
	unsigned char *video;
	video = (unsigned char *) (real_screen+ 2 * x + 160 * y);
	if (c == 10) {			
		x = 0;
		y++;
	} else if (c == 8) {	
		if (x) {
			*(video + 1) = 0x0;
			x--;
		}
	} else if (c == 9) {	
		x = x + 8 - (x % 8);
	} else if (c == 13) {	
		x = 0;
	} else {		
		*video = c;
		*(video + 1) = kattr;

		x++;
		if (x > 79) {
			x = 0;
			y++;
		}
	}
	if (y > 24)
		scrollup(y - 24);
}
```

We also add a usefull and very known method: [printf](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc#L155)

```cpp
/* put a string in screen */
void Io::print(const char *s, ...){
	va_list ap;

	char buf[16];
	int i, j, size, buflen, neg;

	unsigned char c;
	int ival;
	unsigned int uival;

	va_start(ap, s);

	while ((c = *s++)) {
		size = 0;
		neg = 0;

		if (c == 0)
			break;
		else if (c == '%') {
			c = *s++;
			if (c >= '0' && c <= '9') {
				size = c - '0';
				c = *s++;
			}

			if (c == 'd') {
				ival = va_arg(ap, int);
				if (ival < 0) {
					uival = 0 - ival;
					neg++;
				} else
					uival = ival;
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				if (neg)
					print("-%s", buf);
				else
					print(buf);
			}
			 else if (c == 'u') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print(buf);
			} else if (c == 'x' || c == 'X') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 'p') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);
				size = 8;

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 's') {
				print((char *) va_arg(ap, int));
			} 
		} else
			putc(c);
	}

	return;
}
```
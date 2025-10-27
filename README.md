/* * *********************************************************
Programa para leer las aceleraciones Ax, Ay y Az del acelerómetro integrado en la Micro:bit V2
Muestra en la pantalla Led a traves de una matriz 5x5 una flecha que señala hacia abajo dependiendo de la inclinación o posición del sensor. 
Autores: Frida Justine Cervantes Galvan
         Rodrigo Mecalco Galván
         Jesús Eduardo Moratilla Guexpal

* ** ** * * * * * * * * * * * * * * * * * * * * * * * * * */

//Declaración de bibliotecas
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/drivers/sensor.h>
#include <zephyr/sys/printk.h>
#include <zephyr/display/mb_display.h>
#include <math.h>

//Definición del nodo del acelerómetro
#define ACCEL_NODE DT_ALIAS(accel0)
static const struct device *const accel=DEVICE_DT_GET(ACCEL_NODE);

// Valores leídos
struct sensor_value ax, ay, az;

// Puntero al display LED
struct mb_display *disp;

// Variables numéricas para cálculos
double Ax, Ay, Az, Ax2, Ay2, Az2, Axy;
int rc=0;

//arreglo de imagenes a partir de matriz 5x5
static const struct mb_image pos[]={
                                     MB_IMAGE({0,0,0,0,0}, //up  Z neg 0 
                                              {0,0,0,0,0},
                                              {0,0,1,0,0},
                                              {0,0,0,0,0},
                                              {0,0,0,0,0}),
                                     MB_IMAGE({0,0,0,0,0}, // down z pos 1
                                              {0,1,0,1,0},
                                              {0,0,1,0,0},
                                              {0,1,0,1,0},
                                              {0,0,0,0,0}),
                                     MB_IMAGE({0,0,1,0,0}, //Right x pos 2
                                              {0,0,0,1,0},
                                              {1,1,1,1,1},
                                              {0,0,0,1,0},
                                              {0,0,1,0,0}),
                                     MB_IMAGE({0,0,1,0,0}, //Left x neg 3
                                              {0,1,0,0,0},
                                              {1,1,1,1,1},
                                              {0,1,0,0,0},
                                              {0,0,1,0,0}),
                                     MB_IMAGE({0,0,1,0,0}, //Forward y pos 4
                                              {0,1,1,1,0},
                                              {1,0,1,0,1},
                                              {0,0,1,0,0},
                                              {0,0,1,0,0}),
                                     MB_IMAGE({0,0,1,0,0}, //backwards y neg 5
                                              {0,0,1,0,0},
                                              {1,0,1,0,1},
                                              {0,1,1,1,0},
                                              {0,0,1,0,0}),
                                     MB_IMAGE({0,1,1,1,1}, //sec1 6
                                              {0,0,0,1,1},
                                              {0,0,1,0,1},
                                              {0,1,0,0,1},
                                              {1,0,0,0,0}),
                                     MB_IMAGE({1,1,1,1,0}, //sec2 7
                                              {1,1,0,0,0},
                                              {1,0,1,0,0},
                                              {0,0,0,1,0},
                                              {0,0,0,0,1}),
                                     MB_IMAGE({0,0,0,0,1}, //sec3 8
                                              {1,0,0,1,0},
                                              {1,0,1,0,0},
                                              {1,1,0,0,0},
                                              {1,1,1,1,0}),
                                     MB_IMAGE({1,0,0,0,0}, //sec4 9
                                              {0,1,0,0,1},
                                              {0,0,1,0,1},
                                              {0,0,0,1,1},
                                              {0,1,1,1,1}),
                                     MB_IMAGE({1,1,1,1,1}, //Null
                                              {1,0,0,0,1},
                                              {1,0,1,0,1},
                                              {1,0,0,0,1},
                                              {1,1,1,1,1})};



int main(void){
  
  disp = mb_display_get(); //Inicialización del display
  
	if (!device_is_ready(accel))  //Comprobación del acelerómetro
	{
		printk("Acelerometro fuera de servicio :( \n");
	}

	while (1)  //Bucle principal
	{
		rc=sensor_sample_fetch(accel); // Lectura del sensor
		
		if (rc!=0)
		{
			printk("No se pudo leer :( \n");
			goto end;
		}
		rc=sensor_channel_get(accel, SENSOR_CHAN_ACCEL_X, &ax);
		if (rc==0)
		{
			rc=sensor_channel_get(accel, SENSOR_CHAN_ACCEL_Y, &ay);	
		}
		if (rc==0)
		{
			rc=sensor_channel_get(accel, SENSOR_CHAN_ACCEL_Z, &az);	
		}
		
		if (rc!=0)
		{
			printk("No se pudo leer :( \n");
			goto end;
		}
		
                 //Convertir las lecturas
		Ax2=ax.val2;
		Ax=ax.val1+(Ax2/1000000);
		Ay2=ay.val2;
		Ay=ay.val1+(Ay2/1000000);
		Az2=az.val2;
		Az=az.val1+(Az2/1000000);
		//printk("Aceleracion en X: %f m/s² \n", Ax);
		//printk("Aceleracion en Y: %f m/s² \n", Ay);
		//printk("Aceleracion en Z: %f m/s² \n", Az);
		
		//Calcular el ángulo entre ejes X e Y (atan)
		Axy = atan2(Ax,Ay);
		Axy = Axy*(180/3.1416);
		printk("Grados en XY: %f m/s² \n", Axy);
		
		
		//Condiciones Base Z positivo o negativo (arriba/abajo)
		if (Az > 8){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[0],1);
		  k_msleep(20);
		}
		if (Az < -8){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[1],1);
		  k_msleep(20);
		}
		
	         // Movimientos en XY 
		if(Axy < 30 && Axy > -30 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[4],1);
		  k_msleep(20);
		} 
		if(Axy < -60 && Axy > -120 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[3],1);
		  k_msleep(20);
		}
		if(Axy > 60 && Axy < 120 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[2],1);
		  k_msleep(20);
		} 
		if((Axy > 150 || Axy < -150) && (Az < 7 && Az > -7)){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[5],1);
		  k_msleep(20);
		}
		
		 //Sectores intermedios
		if(Axy > 30 && Axy < 60 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[6],1);
		  k_msleep(20);
		} 
		if(Axy > 120 && Axy < 150 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[9],1);
		  k_msleep(20);
		}
		if(Axy < -30 && Axy > -60 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[7],1);
		  k_msleep(20);
		} 
		if(Axy < -120 && Axy > -150 && Az < 7 && Az > -7){
		  mb_display_image(disp,MB_DISPLAY_MODE_SINGLE,20,&pos[8],1);
		  k_msleep(20);
		} 
	      
	        end:

	}

return 0;
}

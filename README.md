#include "udf.h"

#if sys_linux || sys_unix
# define ISNAN isnan
#else
# define ISNAN _isnan
#endif

int npts[3]= { 42, 12, 42};  /* nX, nY & nZ discretisation of external magnetic file data */
real hz = 1.0;	/* frequency */

real xbounds[]= { 1e8, 1e8, 1e8, -1e8, -1e8, -1e8};
static char XYZ_FILENAME[]={"Export_MHD.pts"};
static char MAXWELL_EXP_FILENAME[]={"Export_Maxwell.fld"};
static char MAGNETIC_FIELD_FILENAME[]={"maxwell_bfield.mag"};
 
/* calculate domain bounding box */
void boundingbox(void)
{
 Domain *d;
 Thread *t;
 face_t f;
 Node *v;
 int n;
 
 d = Get_Domain(1);
 
 thread_loop_f(t,d)
  	{
  	if ( THREAD_TYPE(t) != THREAD_F_INTERIOR ) begin_f_loop(f,t)
  		{
  		f_node_loop(f,t,n)
  			{
  			v = F_NODE(f,t,n);
  			xbounds[0] = MIN( xbounds[0], NODE_X(v) );  /* xmin */
  			xbounds[1] = MIN( xbounds[1], NODE_Y(v) );  /* ymin */
  			xbounds[2] = MIN( xbounds[2], NODE_Z(v) );  /* zmin */
  			xbounds[3] = MAX( xbounds[3], NODE_X(v) );  /* xmax */
  			xbounds[4] = MAX( xbounds[4], NODE_Y(v) );  /* ymax */
  			xbounds[5] = MAX( xbounds[5], NODE_Z(v) );  /* zmax */
  			}
  		}
  	end_f_loop(f,t)	
  	} 	
 Message("\nBounds: %f, %f, %f, %f, %f, %f", xbounds[0], xbounds[1], xbounds[2], xbounds[3], xbounds[4], xbounds[5] );
}
 
/* write point file for Maxwell export */
DEFINE_ON_DEMAND(write_box_coord)
{
 int i, j, k;
 FILE *fp_xyz;
 real x,y,z;
 
 boundingbox();
 fp_xyz=fopen(XYZ_FILENAME,"w");
  /* index varies from 0 to npts[], 
      hence nX will be equal to npts[0]+1 (same for nY & nZ) */
  /* loop order to accomodate mhd external data file format:
      See indexing l in Appendix C of Fluent MHD manual */
  for (k=0; k<npts[2]+1; k++)
	{
	for (j=0; j<npts[1]+1; j++)
		{
		for (i=0; i<npts[0]+1; i++)
			{
			x = xbounds[0]+ i*(xbounds[3]-xbounds[0])/npts[0];
			y = xbounds[1]+ j*(xbounds[4]-xbounds[1])/npts[1];
			z = xbounds[2]+ k*(xbounds[5]-xbounds[2])/npts[2];
			fprintf(fp_xyz, "%18.9e%18.9e%18.9e\n", x, y, z);
			}
		}
	}
 fclose(fp_xyz);
}

DEFINE_ON_DEMAND(write_magdata_file)
{
 int i, n, j;
 FILE *fp_mag, *fp_fld;
 char str[1024];
 char *tok =NULL;
 real tokvals[9], temp;
 
 boundingbox();
 n = (npts[0]+1)*(npts[1]+1)*(npts[2]+1);
 fp_mag = fopen(MAGNETIC_FIELD_FILENAME,"w");
 fp_fld = fopen(MAXWELL_EXP_FILENAME,"r");
 
 fprintf(fp_mag,"MAG-DATA");
 fprintf(fp_mag,"\n%d %d %d", npts[0]+1, npts[1]+1, npts[2]+1);
 fprintf(fp_mag,"\n%e %e", xbounds[0], xbounds[3]);
 fprintf(fp_mag,"\n%e %e", xbounds[1], xbounds[4]);
 fprintf(fp_mag,"\n%e %e", xbounds[2], xbounds[5]);
 fprintf(fp_mag,"\n1 %f", hz);
 
 fgets(str, 1024, fp_fld);
 for (i=1; i<n+1; i++)
	{
		j = 0;
		fgets(str, 1024, fp_fld);
		tok = strtok(str, " ");
		while(tok != NULL)
		{
			temp = atof(tok);
			tokvals[j++] = ISNAN(temp) ? 0 : temp;
			tok = strtok(NULL, " ");
		}
	fprintf(fp_mag, "\n%e %e %e %e %e %e", tokvals[3], tokvals[5], tokvals[7], tokvals[4], tokvals[6], tokvals[8]);
	}
 
 fclose(fp_mag);
 fclose(fp_fld);
 
}

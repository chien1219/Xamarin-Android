# Xamarin-Android
This is written at the time that I encounter some problems during my implementation

## The Orientation of an Image showed different from the one it was captured.

There are some circumstances that after I took a picture (ex. in vertical way) but when you see it's original file it appears horizontally as thumbnail.  
The data record in the **EXIF** seems to adjusted by **Auto-Rotate**  

At this moment if you just decode file as following you will get the wrong h/w and orientation.  

```
public Size GetImageSizeService(string imagePath)
{
    Bitmap imageBmp = BitmapFactory.DecodeFile(imagePath);
		return new Size((double)imageBmp.Width, (double)imageBmp.Height);	
}
```

The [ExifInterface](https://developer.xamarin.com/api/member/Android.Media.ExifInterface.GetAttributeInt/p/System.String/System.Int32/) stored the image information details in **Android.Media** including **ExifInterface.TagOrientation**.
![Image1](https://i.imgur.com/QXDw9NA.jpg)

The following reveal the bitmap in its correct orientation and given width / height.

```
 private Bitmap CheckRotateAndResizeImage(string imagePath, float width, float height)
{
    Bitmap originalImage = BitmapFactory.DecodeFile(imagePath);
    Bitmap resizedImage;
    ExifInterface exif = new ExifInterface(imagePath);

    int degree = 0;

    if (exif != null)
    {
        // Get TagOrientation
        int ori = exif.GetAttributeInt(ExifInterface.TagOrientation, (int)Orientation.Undefined);

        // Get degree
        switch (ori)
        {
            case (int)Orientation.Rotate90:
                degree = 90;
                break;
            case (int)Orientation.Rotate180:
                degree = 180;
                break;
            case (int)Orientation.Rotate270:
                degree = 270;
                break;
            default:
                degree = 0;
                break;
        }
    }
    if (degree != 0)
    {
        // Rotate Bitmap
        Matrix m = new Matrix();
        m.PostRotate((float)degree);
        originalImage = Bitmap.CreateBitmap(originalImage, 0, 0, originalImage.Width, originalImage.Height, m, true);
    }

    if (degree == 90 || degree == 270)
    {
        resizedImage = Bitmap.CreateScaledBitmap(originalImage, (int)height, (int)width, false);
    }

    else
    {
        resizedImage = Bitmap.CreateScaledBitmap(originalImage, (int)width, (int)height, false);
    }

    // Recycle Bitmap to Reduce Memory Usage
    if (originalImage != null)
    {
        originalImage.Recycle();
    }

    if (exif != null)
    {
        exif.Dispose();
    }

    return resizedImage;
}
```  
  
  
## Toast Effect  
![](https://www.studytonight.com/android/images/toast-in-android-4.jpg)  
As I mentioned in [Xamarin iOS](https://github.com/chien1219/Xamarin-iOS), the way to implement toast on Android is more simple.  
  
```
using Android.Widget;
using MyProject.Droid.DependencyService;
using MyProject.Interface;

[assembly: Xamarin.Forms.Dependency(typeof(ToastHelperService))]
namespace MyProject.Droid.DependencyService
{
    public class ToastHelperService : IToastHelper
    {
        double screenHeight = Plugin.XamJam.Screen.CrossScreen.Current.Size.Height * 0.8;
        Toast toast = new Toast(Android.App.Application.Context);

        public void LongAlert(string message)
        {
            toast.Cancel();
            toast = Toast.MakeText(Android.App.Application.Context, message, ToastLength.Long);
            toast.SetGravity(Android.Views.GravityFlags.Bottom, 0, (int)screenHeight);
            toast.Show();
        }
        public void ShortAlert(string message)
        {
            toast.Cancel();
            toast = Toast.MakeText(Android.App.Application.Context, message, ToastLength.Short);
            toast.SetGravity(Android.Views.GravityFlags.Bottom, 0, (int)screenHeight);
            toast.Show();
        }
    }
}
```  
The code above is to show a toast message on the 1/5 place of the screen from bottom (using toast.SetGravity()).  
Toast is already implemented in Android.Widget well and customized, the reason I called Cancel() before show is to prevent from double-clicked or multi-clicked, thus, it work as desired!

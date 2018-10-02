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
  
The Interface in Xamarin.Form:  
```
namespace MyProject.Interface
{
    public interface IToastHelper
    {
        void LongAlert(string message);
        void ShortAlert(string message);
    }
}
```
Dependency Service on Droid:  
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
  
  
  ## Tabbed Page Badge  
  This problem confused me for a long time and I tried several approaches.  
  [TabBadge](https://github.com/xabre/xamarin-forms-tab-badge) by **xabre** is the most popular plug-in to resolve this problem  
  Hence in 2.0.0 version there's no solution if the tabbed bar is at the bottom.  
  More clearly, [Can we change this tab bar at bottom position in android ?](https://github.com/xabre/xamarin-forms-tab-badge/issues/48) this issue reveal that if the version of Xamarin.Forms is higher than 3.1, the bottom bar is ```BottomNavigationView``` instead of ```TabLayout```, thus the badge won't show since it could not access the certain View of Tab.  
  Unfortunately, [TabBadge](https://github.com/xabre/xamarin-forms-tab-badge) has not upload yet to support  ```BottomNavigationView``` in current version, however, [Updated solution](https://github.com/GalaxiaGuy/xamarin-forms-tab-badge/blob/6ff5ae3aa423bf56944dcfd667c203725d117e31/Source/Plugin.Badge.Droid/BadgedTabbedPageRenderer.cs) is provided by **GalaxyGuy**  
  More specific, this [Commit](https://github.com/GalaxiaGuy/xamarin-forms-tab-badge/commit/6ff5ae3aa423bf56944dcfd667c203725d117e31) is accounted for such senario. Thus, we have to apply this solution to our project with so-far version by ourselves.  
  
  ```
[assembly: ExportRenderer(typeof(MainBottomTabbedPage), typeof(MyTabBarRenderer))]
namespace MyProject.Droid.Renderers
{
    public class MyTabBarRenderer : TabbedPageRenderer, BottomNavigationView.IOnNavigationItemReselectedListener, BottomNavigationView.IOnNavigationItemSelectedListener
    {
        private bool _isShiftModeSet = false;
        
        public readonly Dictionary<Element, BadgeView> BadgeViews = new Dictionary<Element, BadgeView>();
        public bool IsBottomTabPlacement => (Element != null) && Xamarin.Forms.PlatformConfiguration.AndroidSpecific.TabbedPage.GetToolbarPlacement(Element.OnThisPlatform()) == Xamarin.Forms.PlatformConfiguration.AndroidSpecific.ToolbarPlacement.Bottom;
        
        private BottomNavigationView _bottomNavigationView;

        public MyTabBarRenderer(Context context) : base(context){}

        protected override void OnElementChanged(ElementChangedEventArgs<TabbedPage> e)
        {
            if (IsBottomTabPlacement)
            {
                _bottomNavigationView = ViewGroup.FindChildOfType<BottomNavigationView>();
                if (_bottomNavigationView == null)
                {
                    Console.WriteLine("Plugin.Badge: No BottomNavigationView found. Badge not added.");
                    return;
                }

                for (var i = 0; i < _bottomNavigationView.Menu.Size(); i++)
                {
                    AddTabBadge(i);
                }
            }
        }

	private void AddTabBadge(int tabIndex)
        {
            var element = Element.Children[tabIndex];
            BadgeView badgeView = null;
            Android.Views.View view = null;
            if (IsBottomTabPlacement)
            {
                view = ((ViewGroup)_bottomNavigationView?.GetChildAt(0))?.GetChildAt(tabIndex);
            }
            badgeView = (view as ViewGroup)?.FindChildOfType<BadgeView>();

            if (badgeView == null)
            {
                var imageView = (view as ViewGroup)?.FindChildOfType<ImageView>();

                var badgeTarget = imageView?.Drawable != null
                    ? (Android.Views.View)imageView
                    : (view as ViewGroup)?.FindChildOfType<TextView>();
                
                //create badge for tab
                badgeView = new BadgeView(Context, badgeTarget);
                
		// Fix the size and optimize
                badgeView.SetMinimumHeight(35);
                badgeView.SetMinimumWidth(35);
                badgeView.TranslationY = 20;
                badgeView.SetTextSize(Android.Util.ComplexUnitType.Dip, 5);
            }

            BadgeViews[element] = badgeView;
            
            badgeView.UpdateFromElement(element);
        
            element.PropertyChanged -= OnTabbedPagePropertyChanged;
            element.PropertyChanged += OnTabbedPagePropertyChanged;
        }

        protected virtual void OnTabbedPagePropertyChanged(object sender, System.ComponentModel.PropertyChangedEventArgs e)
        {
            if (!(sender is Element element))
            {
                return;
            }

            if (BadgeViews.TryGetValue(element, out var badgeView))
            {
                badgeView.UpdateFromPropertyChangedEvent(element, e);
            }
        }
    }
}
  ```
  
  And for the extension(if we override those function in our renderer, the original extension of plugin won't be accessible)
  we add there lines followed by AddBadge():  
  
  ```
        public void UpdateFromElement(BadgeView badgeView, Page element)
        {
            //get text
            var badgeText = TabBadge.GetBadgeText(element);
            badgeView.Text = badgeText;

            // set color if not default
            var tabColor = TabBadge.GetBadgeColor(element);
            if (tabColor != Color.Default)
            {
                badgeView.BadgeColor = tabColor.ToAndroid();
            }

            // set text color if not default
            var tabTextColor = TabBadge.GetBadgeTextColor(element);
            if (tabTextColor != Color.Default)
            {
                badgeView.TextColor = tabTextColor.ToAndroid();
            }

            // set font if not default
            var font = TabBadge.GetBadgeFont(element);
            if (font != Font.Default)
            {
                badgeView.Typeface = font.ToTypeface();
            }

            var margin = TabBadge.GetBadgeMargin(element);
            badgeView.SetMargins((float)margin.Left, (float)margin.Top, (float)margin.Right, (float)margin.Bottom);

            // set position
            badgeView.Postion = TabBadge.GetBadgePosition(element);
        }

        public void UpdateFromPropertyChangedEvent(BadgeView badgeView, Element element, PropertyChangedEventArgs e)
        {
            if (e.PropertyName == TabBadge.BadgeTextProperty.PropertyName)
            {
                badgeView.Text = TabBadge.GetBadgeText(element);
                return;
            }

            if (e.PropertyName == TabBadge.BadgeColorProperty.PropertyName)
            {
                badgeView.BadgeColor = TabBadge.GetBadgeColor(element).ToAndroid();
                return;
            }

            if (e.PropertyName == TabBadge.BadgeTextColorProperty.PropertyName)
            {
                badgeView.TextColor = TabBadge.GetBadgeTextColor(element).ToAndroid();
                return;
            }

            if (e.PropertyName == TabBadge.BadgeFontProperty.PropertyName)
            {
                badgeView.Typeface = TabBadge.GetBadgeFont(element).ToTypeface();
                return;
            }

            if (e.PropertyName == TabBadge.BadgePositionProperty.PropertyName)
            {
                badgeView.Postion = TabBadge.GetBadgePosition(element);
                return;
            }

            if (e.PropertyName == TabBadge.BadgeMarginProperty.PropertyName)
            {
                var margin = TabBadge.GetBadgeMargin(element);
                badgeView.SetMargins((float)margin.Left, (float)margin.Top, (float)margin.Right, (float)margin.Bottom);
                return;
            }
        }
  ```  
  In order to tine the icon (if you custom your renderer or inherent from **BadgedTabbedPageRenderer**, it is possible to lose your tint when selected):  
  ```
   bool BottomNavigationView.IOnNavigationItemSelectedListener.OnNavigationItemSelected(IMenuItem item)
        {
            if (Element is MainBottomTabbedPage)
            {
                for (var i = 0; i < _bottomNavigationView.Menu.Size(); i++)
                {
                    _bottomNavigationView.Menu.GetItem(i).Icon.SetColorFilter(Android.Graphics.Color.White, Android.Graphics.PorterDuff.Mode.Multiply);
                }
                item.Icon.SetColorFilter(Android.Graphics.Color.DodgerBlue, Android.Graphics.PorterDuff.Mode.Multiply);
            }
             return base.OnNavigationItemSelected(item);
        }
  ```
  
  There are also many ways to handle Tab Badge, such as:  
    
  1. [Draw the badge in Android.](https://www.xamboy.com/2018/03/08/adding-badge-to-toolbaritem-in-xamarin-forms/)  
  but there are also some problem with it, the badge circle will not appear out of the icon image, which means that it will be clipped and made a quater circle.  
  2. Change icon image on cross platform.
  There are also some problem with this, the image can't be change dynamically on Xamarin.Forms for iOS, so the solution may be added in iOS renderer:  
  ```
  public override void ViewWillAppear(bool animated)
  {
  	base.ViewWillAppear(animated);
  	foreach (UITabBarItem item in TabBar.Items)
  	{
	    //This line is in order to dismiss the label text.
  	    item.ImageInsets = new UIEdgeInsets(top: 6, left: 0, bottom: -6, right: 0);
  	    //Change image
	    item.Image = new UIImage("Image.png");
	    //Keep original color without tint
	    item.Image = item.Image.ImageWithRenderingMode(UIKit.UIImageRenderingMode.AlwaysOriginal);
  	}
  }
  ```  
  
  But this is more complicated and also come out with tint problem on Android (which means that the icon will not be shown as original color but tint one.)  
  
### 2018/9/17 Update
  **xabre** has update solution to bottom tabpage for Android.  
  But there are still some problem with it, for example, it could not render with custom renderer.  
  

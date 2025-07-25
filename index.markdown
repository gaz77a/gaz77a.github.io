---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<div style="
  max-width: 1100px;
  width: 100%;
  margin: 42px auto 44px auto;
  border-radius: 28px;
  box-shadow:
    0 20px 80px -4px rgba(44,60,180,0.72),             /* bold shadow */
    0 28px 128px 0 rgba(44,50,100,0.19);               /* halo */
  overflow: visible;        /* shadow spills out, not clipped */
  background: none;
  position: relative;
  display: block;
">
    <div style="position:relative;background:linear-gradient(91deg,#1F2937 0%, #4F8A8B 100%);color:#fff;padding:2.5em 1em 2em 1em;border-radius:1em;box-shadow:0 6px 24px rgba(79,138,139,0.14);text-align:center;margin-bottom:2.5em;overflow:hidden;">
    <h1 style="font-size:2.9em;letter-spacing:2px;font-weight:900;margin-top:0;">

        Step Into Full Stack,<br> <span style="color:#4F8A8B;background:rgba(255,255,255,0.06);border-radius:0.4em;padding:0 0.4em;">Full Beard Blogs</span>

    </h1>

    </div>

</div>
<script>
  // Simple shimmer effect for the button (works on most modern browsers)
  document.querySelectorAll("a .shine").forEach(function(el){
    let parent = el.parentElement;
    parent.addEventListener("mouseenter", function(){
      el.style.transform="translateX(0)";
    });
    parent.addEventListener("mouseleave", function(){
      el.style.transform="translateX(-100%)";
    });
  });
</script>

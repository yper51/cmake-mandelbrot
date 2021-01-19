cmake_minimum_required(VERSION 2.8.11) # timestamp requires (>=2.8.11)
project(cmake-mandelbrot LANGUAGES NONE)

function(env_default key value)
    if(NOT DEFINED ${key})
        set(${key} ${value} PARENT_SCOPE)
    endif()
endfunction()

env_default(max_iter 100)
env_default(gradient " .-=+*#%@")

env_default(y_begin -1.3)
env_default(y_end 1.3)
env_default(y_step 0.1)

env_default(x_begin -2.1)
env_default(x_end 1.1)
env_default(x_step 0.04)

option(timed "timed" OFF)

# used to convert back the frac digits on gcvt
set(fract_scale 100000)
string(LENGTH ${fract_scale} fract_max_length)
math(EXPR fract_max_length "${fract_max_length} - 1")

# can't use all of the 64 bits because of RANGE limitations
# string -> real
function(real result x)
    string(REPLACE "." ";" number ${x})
    list(GET number 0 int_part)
    list(GET number 1 frac_part)
    string(SUBSTRING ${int_part} 0 1 sign)
    string(SUBSTRING ${frac_part} 0 ${fract_max_length} frac_part)
    string(LENGTH ${frac_part} frac_length)
    string(REPEAT "0" ${frac_length} zeroes)
    
    if(NOT ${sign} STREQUAL "-")
        set(sign "+")
    endif()

    math(EXPR tmp "(${int_part} << 16) ${sign} (${frac_part} << 16) / 1${zeroes}")

    set(${result} ${tmp} PARENT_SCOPE)
endfunction()

# PUSH ---- debug purposes only ----
function(abs x out)
    math(EXPR mask "${x} >> 63")
    math(EXPR tmp "${mask} ^ (${mask} + ${x})")
    
    set(${out} ${tmp} PARENT_SCOPE)
endfunction()

# real -> string
function(gcvt x out)
    string(SUBSTRING ${x} 0 1 sign)
    if(NOT ${sign} STREQUAL "-")
        set(sign "")
    endif()
    abs(${x} abs_x)
    math(EXPR int_part "${abs_x} >> 16")
    math(EXPR frac_part "((${abs_x} & 0xFFFF) * ${fract_scale}) >> 16")
    string(LENGTH ${frac_part} frac_length)
    math(EXPR zero_count "${fract_max_length} - ${frac_length}")
    string(REPEAT "0" ${zero_count} zeros)
    
    set(${out} "${sign}${int_part}.${zeros}${frac_part}" PARENT_SCOPE)
endfunction()
# POP ---- debug purposes only ----

function(fmul lhs rhs result)
    math(EXPR tmp "(${lhs} * ${rhs}) >> 16")
    set(${result} ${tmp} PARENT_SCOPE)
endfunction()

string(REGEX MATCHALL "." gradient ${gradient})
list(LENGTH gradient gradient_length)
math(EXPR gradient_length "${gradient_length} - 1")

real(four 4.0)

real(y_begin ${y_begin})
real(y_end ${y_end})
real(y_step ${y_step})

real(x_begin ${x_begin})
real(x_end ${x_end})
real(x_step ${x_step})

gcvt(${y_begin} test_out)
message("begin ${y_begin} ${test_out}")

if(timed)
    string(TIMESTAMP start_timer "%s")
endif()

foreach(y RANGE ${y_begin} ${y_end} ${y_step})
    set(line "")
    foreach(x RANGE ${x_begin} ${x_end} ${x_step})
        set(zi 0)
        set(zr 0)
        set(iter 0) # if not found
        foreach(i RANGE ${max_iter})
            fmul(${zr} ${zr} zr_sq)
            fmul(${zi} ${zi} zi_sq)
            math(EXPR temp "${zi_sq} + ${zr_sq}")
            if (${temp} GREATER_EQUAL ${four})
                set(iter ${i})
                break()
            endif()
            fmul(${zr} ${zi} zr_zi)
            math(EXPR zr_local "${zr_sq} - ${zi_sq} + ${x}")
            math(EXPR zi_local "2 * ${zr_zi} + ${y}")
            set(zi ${zi_local})
            set(zr ${zr_local})
        endforeach()
        math(EXPR index "${iter} % ${gradient_length}")
        list(GET gradient ${index} curr_char)
        string(APPEND line ${curr_char})
    endforeach()
    message(${line})
endforeach()

if(timed)
    string(TIMESTAMP end_timer "%s")
    math(EXPR time "${end_timer} - ${start_timer}")
    message("${time} seconds taken.")
endif()

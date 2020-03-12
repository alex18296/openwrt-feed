SRC_DIR := ./src

CFLAGS += -DL_ENDIAN -O3 -DNDEBUG -fPIC -O2 -Werror -Wall -Wno-unused-parameter -Wno-unused-function -Wno-missing-braces -ggdb -std=gnu90 -I $(SRC_DIR) -c
LFLAGS += -fPIC -O3 -DNDEBUG  -shared -Wl,-soname,gost.so

OBJ_LIST := gost_ec_keyx.o gost_ec_sign.o gost89.o gosthash.o gost_grasshopper_core.o gost_grasshopper_defines.o \
	gost_grasshopper_galois_precompiled.o gost_grasshopper_precompiled.o gost_grasshopper_cipher.o gost_grasshopper_mac.o \
	gosthash2012.o e_gost_err.o gost_asn1.o gost_crypt.o gost_ctl.o gost_eng.o gost_keywrap.o gost_params.o gost_ameth.o \
	gost_md.o gost_md2012.o gost_pmeth.o

all: libgost

libgost: $(OBJ_LIST)
	$(CC) $(LFLAGS) -o gost.so $(OBJ_LIST) -lcrypto

%.o: $(SRC_DIR)/%.c
	$(CC) $(CFLAGS) -o $@ $<

clean:
	rm *.o gost.so
